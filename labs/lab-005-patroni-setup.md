# Lab 005 — Patroni HA Cluster

**Topics**: 5.1–5.4 HA with Patroni, etcd, HAProxy
**Read first**: `curriculum/5-ha/5.1-ha-concepts.md` through `curriculum/5-ha/5.4-haproxy-vip.md`
**Time**: 3–4 hours (this is a full cluster build)
**Requires**: 3 servers/VMs

```
pg-node1  (10.0.0.10)  — PostgreSQL + Patroni
pg-node2  (10.0.0.20)  — PostgreSQL + Patroni
etcd-node (10.0.0.30)  — etcd DCS
```

Replace IPs with your actual values throughout.

A single etcd node has no quorum to lose: fine for learning Patroni, never for production (5.3 explains why you need three or more, and 5.5 where to put them).

---

## Task 1: Install and Start etcd

```bash
# On etcd-node (10.0.0.30). Debian 12 / Ubuntu 24.04 package names;
# older releases: apt install -y etcd
apt install -y etcd-server etcd-client

cat > /etc/default/etcd << 'EOF'
ETCD_NAME="etcd-node1"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_LISTEN_CLIENT_URLS="http://10.0.0.30:2379,http://127.0.0.1:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://10.0.0.30:2379"
ETCD_LISTEN_PEER_URLS="http://10.0.0.30:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.0.0.30:2380"
ETCD_INITIAL_CLUSTER="etcd-node1=http://10.0.0.30:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="pg-ha"
EOF

systemctl enable etcd && systemctl start etcd

# Verify
etcdctl --endpoints=http://10.0.0.30:2379 endpoint health
```

---

## Task 2: Install Patroni on Both PostgreSQL Nodes

```bash
# On BOTH pg-node1 and pg-node2
apt install -y postgresql-16 python3-venv

# Patroni in a virtualenv (recent Debian/Ubuntu refuse a system-wide pip install).
# Patroni doesn't bring a PostgreSQL driver by itself, so psycopg3 is named in the extras.
python3 -m venv /opt/patroni
/opt/patroni/bin/pip install 'patroni[psycopg3,etcd3]'
ln -s /opt/patroni/bin/patroni /opt/patroni/bin/patronictl /usr/local/bin/
# (Alternative: apt install patroni, from the PostgreSQL apt repository.)

# Stop and disable the packaged PostgreSQL service — Patroni manages PostgreSQL now
systemctl stop postgresql
systemctl disable postgresql

# Installing postgresql-16 also created a cluster in /var/lib/postgresql/16/main.
# Remove it: pg-node1 must bootstrap a fresh cluster and pg-node2 must clone pg-node1
# (an unrelated existing cluster makes Patroni stop with "system ID mismatch").
pg_dropcluster --stop 16 main
```

---

## Task 3: Create Patroni Config

```bash
# On pg-node1 (10.0.0.10)
mkdir -p /etc/patroni

cat > /etc/patroni/patroni.yml << 'EOF'
scope: pg-cluster
namespace: /service/
name: pg-node1                          # UNIQUE per node

restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.0.0.10:8008       # THIS node's IP

etcd3:
  hosts: 10.0.0.30:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        max_connections: 200
        max_wal_senders: 10
        max_replication_slots: 10
        wal_log_hints: "on"

  initdb:
    - encoding: UTF8
    - data-checksums

  pg_hba:
    - local all all peer
    - host all all 127.0.0.1/32 scram-sha-256
    - host all all ::1/128 scram-sha-256
    - host replication replicator 10.0.0.0/24 scram-sha-256
    - host all all 10.0.0.0/24 scram-sha-256    # lab subnet only, never 0.0.0.0/0

  # No bootstrap.users section: Patroni 4 removed it. Patroni creates the
  # superuser, replicator and rewind users from "authentication" below.

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.0.0.10:5432       # THIS node's IP
  data_dir: /var/lib/postgresql/16/main
  bin_dir: /usr/lib/postgresql/16/bin

  authentication:
    replication:
      username: replicator
      password: replpass123
    superuser:
      username: postgres
      password: pgpass123
    rewind:
      username: rewind_user
      password: rewindpass123

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
EOF

# It contains passwords, and Patroni runs as postgres
chown postgres: /etc/patroni/patroni.yml
chmod 600 /etc/patroni/patroni.yml
```

```bash
# On pg-node2: same file but change these two lines:
# name: pg-node2
# connect_address: 10.0.0.20:8008 (restapi section)
# connect_address: 10.0.0.20:5432 (postgresql section)
```

---

## Task 4: Create systemd Service and Start Patroni

```bash
# On BOTH nodes

cat > /etc/systemd/system/patroni.service << 'EOF'
[Unit]
Description=Patroni PostgreSQL HA
After=syslog.target network.target

[Service]
Type=simple
User=postgres
ExecStart=/usr/local/bin/patroni /etc/patroni/patroni.yml
KillMode=process
TimeoutSec=30
Restart=no

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable patroni

# Start pg-node1 FIRST (it bootstraps the cluster)
# On pg-node1:
systemctl start patroni
journalctl -u patroni -f     # Watch until you see "promoted self to leader"

# Then start pg-node2:
systemctl start patroni
journalctl -u patroni -f     # Watch until you see "a secondary, and following a leader (pg-node1)"
```

---

## Task 5: Verify the Cluster

```bash
# Check cluster status
patronictl -c /etc/patroni/patroni.yml list

# Expected output:
# + Cluster: pg-cluster ----+----+-----------+
# | Member   | Host          | Role    | State   | TL | Lag in MB |
# | pg-node1 | 10.0.0.10:... | Leader  | running |  1 |           |
# | pg-node2 | 10.0.0.20:... | Replica | running |  1 |         0 |
```

```sql
-- Test write on Leader
psql -h 10.0.0.10 -U postgres -c "CREATE TABLE ha_test (id serial, msg text);"
psql -h 10.0.0.10 -U postgres -c "INSERT INTO ha_test (msg) VALUES ('cluster is alive');"

-- Test read on Replica
psql -h 10.0.0.20 -U postgres -c "SELECT * FROM ha_test;"
-- Must see the inserted row
```

```bash
# Check Patroni REST API (used by HAProxy for health checks)
curl http://10.0.0.10:8008/primary    # should return HTTP 200 on Leader
curl http://10.0.0.20:8008/primary    # should return HTTP 503 on Replica
curl http://10.0.0.20:8008/replica    # should return HTTP 200 on Replica
```

---

## Task 6: Automatic Failover Test

```bash
# First, prove what does NOT cause a failover. On pg-node1, stop only PostgreSQL:
sudo -u postgres /usr/lib/postgresql/16/bin/pg_ctl stop -D /var/lib/postgresql/16/main -m immediate
# Watch for ~20 seconds: Patroni still holds the leader lock, so it simply starts
# PostgreSQL again. That's the right behavior: restart it, don't fail over.
watch -n 1 "patronictl -c /etc/patroni/patroni.yml list"

# Now simulate a crash of the whole node. Best: hard power-off the pg-node1 VM from
# the hypervisor. Next best, on pg-node1: kill Patroni and all PostgreSQL processes
# at once (both run as the postgres user):
sudo pkill -9 -u postgres

# Watch failover from pg-node2:
watch -n 1 "patronictl -c /etc/patroni/patroni.yml list"

# Expected sequence (ttl = 30):
# t+0:            pg-node1 stops answering
# by about t+30:  its leader key expires and pg-node2 acquires it
# moments later:  pg-node2 promoted to Leader

# Record the failover time: _____ seconds
```

```sql
-- Verify pg-node2 is now primary and accepts writes
psql -h 10.0.0.20 -U postgres -c "INSERT INTO ha_test (msg) VALUES ('after failover');"
psql -h 10.0.0.20 -U postgres -c "SELECT pg_is_in_recovery();"  -- must return false
```

```bash
# Bring pg-node1 back — Patroni will pg_rewind and rejoin as Replica
# On pg-node1 (if you powered the VM off, just boot it: the service starts by itself):
systemctl start patroni
sleep 30
patronictl -c /etc/patroni/patroni.yml list
# pg-node1 should now show as Replica
```

---

## Task 7: Install HAProxy

```bash
# On pg-node1 (or a separate server)
apt install -y haproxy

cat > /etc/haproxy/haproxy.cfg << 'EOF'
global
    maxconn 200

defaults
    mode tcp
    option tcplog
    timeout connect 5s
    timeout client 30m
    timeout server 30m

# Primary-only: write traffic (port 5000)
listen postgres_write
    bind *:5000
    option httpchk GET /primary
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pg-node1 10.0.0.10:5432 check port 8008
    server pg-node2 10.0.0.20:5432 check port 8008

# Replicas: read traffic (port 5001)
listen postgres_read
    bind *:5001
    balance roundrobin
    option httpchk GET /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pg-node1 10.0.0.10:5432 check port 8008
    server pg-node2 10.0.0.20:5432 check port 8008

listen stats
    bind *:7000
    stats enable
    stats uri /
    stats refresh 5s
EOF

systemctl enable haproxy && systemctl start haproxy
haproxy -c -f /etc/haproxy/haproxy.cfg     # verify config is valid
```

```sql
-- Test HAProxy routing
-- Port 5000 → should always go to current Leader
psql -h 10.0.0.10 -p 5000 -U postgres -c "SELECT pg_is_in_recovery();"  -- false
psql -h 10.0.0.10 -p 5000 -U postgres -c "INSERT INTO ha_test (msg) VALUES ('via haproxy');"

-- Port 5001 → should go to Replica
psql -h 10.0.0.10 -p 5001 -U postgres -c "SELECT pg_is_in_recovery();"  -- true
```

---

## Task 8: HAProxy Survives Failover

```bash
# 1. From the client machine, start continuous writes through HAProxy port 5000:
for i in $(seq 1 200); do
    psql -h 10.0.0.10 -p 5000 -U postgres \
        -c "INSERT INTO ha_test (msg) VALUES ('write $i');" 2>&1
    sleep 0.5
done

# 2. While that runs (from another terminal), crash the current Leader. It must not
#    be the node running HAProxy (after Task 6 the Leader is pg-node2). Power it off,
#    or on that node:
sudo pkill -9 -u postgres

# How many writes failed during failover? ___
# How long was the outage window? ___
```

---

## Completion Checklist

- [ ] etcd installed and healthy
- [ ] Patroni installed on both PostgreSQL nodes
- [ ] Cluster bootstrapped (pg-node1=Leader, pg-node2=Replica)
- [ ] Data written on Leader visible on Replica
- [ ] Patroni REST API returns correct status codes per role
- [ ] Automatic failover tested: pg-node1 killed, pg-node2 promoted
- [ ] Recorded actual failover time in seconds
- [ ] pg-node1 rejoined as Replica after recovery
- [ ] HAProxy installed and routing writes to Leader, reads to Replica
- [ ] HAProxy failover tested: writes continue (with brief outage) through HAProxy during failover
- [ ] Updated `progress.md`

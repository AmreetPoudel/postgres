# T-14 — PgBouncer Queue Buildup: Connections Waiting in Pool

**Frequency**: Common on traffic spikes
**Severity**: P1 — request latency explodes, clients timeout
**Related**: 2.4 PgBouncer

---

## The Symptom

Application latency spikes from 20ms to 5000ms. Monitoring shows PgBouncer has many connections in "waiting" state. PostgreSQL itself is fine — only a few connections active.

---

## Why This Happens

In transaction pooling mode, PgBouncer opens up to `default_pool_size` server connections per database/user pair (a `pool_size=` on the database's line overrides it). If more clients arrive than pool_size, they queue. If the queue grows faster than transactions complete → latency explodes.

---

## Diagnosis

```bash
# Connect to PgBouncer admin console
psql -h 127.0.0.1 -p 6432 -U pgbouncer pgbouncer

# See all pools
SHOW POOLS;
# cl_waiting > 0 means clients are queued — RED FLAG
# cl_active  = clients running a query
# sv_active  = server connections busy
# sv_idle    = server connections ready

# Current client connections
SHOW CLIENTS;

# What's slow?
SHOW STATS;
```

---

## Fix Options

**Immediate**: Increase the pool temporarily (the setting is `default_pool_size`; there's no global `pool_size`):
```ini
# pgbouncer.ini (was 20)
default_pool_size = 50
```
```bash
psql -h 127.0.0.1 -p 6432 -U pgbouncer pgbouncer -c "RELOAD;"
```

**Investigate long transactions** eating pool slots:
```sql
-- On PostgreSQL: find long-running queries
SELECT pid, now() - query_start AS duration, state, left(query,100)
FROM pg_stat_activity WHERE state = 'active' ORDER BY duration DESC;
```

**Set `query_wait_timeout`** so queued clients fail fast instead of piling up (`client_login_timeout` only covers the login handshake):
```ini
; seconds a client may wait for a server connection (default 120)
query_wait_timeout = 10
; cancel queries running longer than this (default 0 = off)
query_timeout = 30
```

---

## Lab Exercise

```bash
# Requires PgBouncer installed (apt install pgbouncer). This lab runs a second,
# throwaway instance on port 6433, so it can't disturb a real one.

# A role for PgBouncer to log in as. PgBouncer passes the password from its
# auth_file on to PostgreSQL, so it has to be the real one.
psql -U postgres -c "CREATE ROLE bouncer_lab LOGIN PASSWORD 'labpass';"

# default_pool_size = 2 is tiny on purpose, to demonstrate queuing
cat > /tmp/pgbouncer_lab.ini << 'EOF'
[databases]
labdb = host=127.0.0.1 port=5432 dbname=postgres

[pgbouncer]
listen_port       = 6433
listen_addr       = 127.0.0.1
auth_type         = scram-sha-256
auth_file         = /tmp/pgbouncer_users.txt
pool_mode         = transaction
default_pool_size = 2
max_client_conn   = 100
admin_users       = pgadmin
logfile           = /tmp/pgbouncer_lab.log
pidfile           = /tmp/pgbouncer_lab.pid
EOF

cat > /tmp/pgbouncer_users.txt << 'EOF'
"bouncer_lab" "labpass"
"pgadmin" "adminpass"
EOF

# PgBouncer refuses to run as root: start it as your normal user (or postgres)
pgbouncer -d /tmp/pgbouncer_lab.ini
```

```bash
# Simulate overload: 10 concurrent clients through a pool of 2
export PGPASSWORD=labpass
for i in $(seq 1 10); do
    psql -h 127.0.0.1 -p 6433 -U bouncer_lab labdb \
        -c "SELECT pg_sleep(2); SELECT $i AS id;" &
done

# Immediately check queue buildup
sleep 1
PGPASSWORD=adminpass psql -h 127.0.0.1 -p 6433 -U pgadmin pgbouncer -c "SHOW POOLS;"
# cl_waiting should be > 0

wait
```

```bash
# Increase the pool and compare: set default_pool_size = 10 in the ini, then
PGPASSWORD=adminpass psql -h 127.0.0.1 -p 6433 -U pgadmin pgbouncer -c "RELOAD;"
# ...and repeat the loop

# Clean up: stop only this instance (never a blanket pkill pgbouncer)
kill "$(cat /tmp/pgbouncer_lab.pid)"
psql -U postgres -c "DROP ROLE bouncer_lab;"
```

**Write down**: With pool_size=2, max cl_waiting seen: ___  
With pool_size=10, max cl_waiting seen: ___

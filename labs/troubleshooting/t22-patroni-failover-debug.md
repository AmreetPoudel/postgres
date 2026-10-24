# T-22 — Patroni Failover Didn't Happen (Debugging a Stuck Cluster)

**Frequency**: Occasional (hardware + configuration dependent)
**Severity**: P0 — primary is down, no automatic failover
**Related**: 5.2 Patroni, 5.3 etcd

---

## The Symptom

Primary PostgreSQL crashes. Expected: standby promotes within 30 seconds. Actual: cluster stays leaderless for minutes. Manual intervention required.

---

## Why This Happens

1. **etcd quorum lost** — if etcd nodes are down, Patroni can't elect a new leader
2. **nofailover: true** — node tagged to not take over
3. **maximum_lag_on_failover** exceeded — standby too far behind to be considered safe
4. **pg_rewind disabled** — old primary can't rejoin, Patroni waits
5. **Patroni process itself crashed** on standby

---

## Diagnosis

```bash
# 1. Is Patroni running on standby?
systemctl status patroni
journalctl -u patroni -n 50 --no-pager

# 2. Is etcd reachable and healthy?
etcdctl --endpoints=http://10.0.0.30:2379 endpoint health
etcdctl --endpoints=http://10.0.0.30:2379 get /service/pg-cluster/leader

# 3. What does Patroni see?
patronictl -c /etc/patroni/patroni.yml list

# 4. Check nofailover tag
patronictl -c /etc/patroni/patroni.yml show-config | grep nofailover

# 5. Check replication lag at time of failure
# (from monitoring data or pg_stat_replication before crash)
```

---

## Fix: Manual Promotion

```bash
# If Patroni is running but won't auto-promote:
patronictl -c /etc/patroni/patroni.yml failover pg-cluster --candidate pg-node2 --force

# If Patroni is not responding, promote PostgreSQL manually:
pg_ctl promote -D /var/lib/postgresql/16/main

# Or via psql (PG12+):
psql -c "SELECT pg_promote();"
```

---

## Fix: etcd Quorum Lost

```bash
# Check which etcd nodes are alive
for ep in http://10.0.0.1:2379 http://10.0.0.2:2379 http://10.0.0.3:2379; do
    echo -n "$ep: "
    etcdctl --endpoints=$ep endpoint health 2>&1 | head -1
done

# If quorum lost (less than half alive), you must restore etcd first
# Or temporarily point Patroni to surviving etcd nodes:
# Edit patroni.yml: etcd3.hosts = surviving-node:2379
systemctl restart patroni
```

---

## Lab Exercise (requires Patroni cluster from Lab 005)

```bash
# Force a scenario where Patroni must decide:
# Stop Patroni (not PostgreSQL) on the primary node
# On pg-node1:
systemctl stop patroni

# PostgreSQL keeps running but Patroni won't renew the etcd lease
# Watch what happens after TTL expires
watch -n 1 "patronictl -c /etc/patroni/patroni.yml list"
```

```bash
# Expected: pg-node2 promotes after TTL seconds
# If it doesn't:
journalctl -u patroni -n 50 --no-pager    # on pg-node2

# Check etcd leader key
etcdctl --endpoints=http://10.0.0.30:2379 get /service/pg-cluster/leader

# Force it:
patronictl -c /etc/patroni/patroni.yml failover pg-cluster --force
```

```bash
# Restore: bring pg-node1 back as replica
systemctl start patroni
sleep 15
patronictl -c /etc/patroni/patroni.yml list
```

**Write down**: Did auto-failover work? ___  
If not, what did the logs say? ___

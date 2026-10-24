# T-22 — Patroni Failover Didn't Happen (Debugging a Stuck Cluster)

**Frequency**: Occasional (hardware + configuration dependent)
**Severity**: P0 — primary is down, no automatic failover
**Related**: 5.2 Patroni, 5.3 etcd

---

## The Symptom

Primary PostgreSQL crashes. Expected: standby promotes within 30 seconds. Actual: cluster stays leaderless for minutes. Manual intervention required.

---

## Why This Happens

1. **The old leader isn't actually gone** — Patroni on the primary is alive and keeps renewing its lease; it just restarts a crashed PostgreSQL. That's a restart, not a failover, and it's correct
2. **etcd quorum lost** — no majority, no leader election
3. **The cluster is paused** — `patronictl pause` (maintenance mode) stops all automatic actions; `patronictl list` shows "Maintenance mode: on"
4. **No eligible replica** — every candidate is tagged `nofailover: true`, lags more than `maximum_lag_on_failover`, or isn't running and streaming
5. **Synchronous mode** — with `synchronous_mode` on, only the current synchronous standby may promote itself; if it's gone too, Patroni waits rather than lose acknowledged commits
6. **Patroni itself crashed or was stopped** on the replica

---

## Diagnosis

```bash
# 1. Is Patroni running on standby?
systemctl status patroni
journalctl -u patroni -n 50 --no-pager

# 2. Is etcd reachable and healthy?
etcdctl --endpoints=http://10.0.0.30:2379 endpoint health
etcdctl --endpoints=http://10.0.0.30:2379 get /service/pg-cluster/leader

# 3. What does Patroni see? (look for "Maintenance mode: on", and the Tags column)
patronictl -c /etc/patroni/patroni.yml list

# 4. Tags (nofailover, nosync...) are per node, in each node's own patroni.yml,
#    not in the shared config
grep -A5 '^tags:' /etc/patroni/patroni.yml
# Shared settings: synchronous_mode, maximum_lag_on_failover, ttl...
patronictl -c /etc/patroni/patroni.yml show-config

# 5. Check replication lag at time of failure
# (from monitoring data or pg_stat_replication before crash)
```

---

## Fix: Manual Promotion

```bash
# If Patroni is running but won't auto-promote (and you've understood why, above):
patronictl -c /etc/patroni/patroni.yml failover pg-cluster --candidate pg-node2 --force

# If the cluster is paused, resume it instead:
patronictl -c /etc/patroni/patroni.yml resume

# Promoting PostgreSQL behind Patroni's back is a last resort, and only after making
# sure the old primary is down or fenced off (5.1). Stop Patroni on this node first,
# or it will fight you:
systemctl stop patroni                      # this stops PostgreSQL too
sudo -u postgres /usr/lib/postgresql/16/bin/pg_ctl start -D /var/lib/postgresql/16/main
sudo -u postgres /usr/lib/postgresql/16/bin/pg_ctl promote -D /var/lib/postgresql/16/main
```

---

## Fix: etcd Quorum Lost

```bash
# Check which etcd nodes are alive
for ep in http://10.0.0.1:2379 http://10.0.0.2:2379 http://10.0.0.3:2379; do
    echo -n "$ep: "
    etcdctl --endpoints=$ep endpoint health 2>&1 | head -1
done

# Quorum lost (fewer than a majority alive)? Pointing Patroni at a surviving member
# doesn't help: a member without quorum can't serve writes. Bring the missing members
# back. If they're gone for good, rebuild etcd from a survivor as a new one-member
# cluster (etcd --force-new-cluster, or restore a snapshot) and add fresh members.
# Patroni recovers as soon as etcd has a quorum again.
```

---

## Lab Exercise (requires Patroni cluster from Lab 005)

```bash
# Scenario 1: stop Patroni cleanly on the primary (on pg-node1)
systemctl stop patroni
# Patroni shuts PostgreSQL down and releases the leader key, so pg-node2 takes over
# within seconds, without waiting for the TTL. Watch:
watch -n 1 "patronictl -c /etc/patroni/patroni.yml list"
```

```bash
# Scenario 2: Patroni dies but PostgreSQL keeps running. This is what the watchdog
# exists for (5.1). Make pg-node1 the Leader again first (patronictl switchover),
# then on pg-node1:
sudo pkill -9 -f 'bin/patroni'
# PostgreSQL on pg-node1 is still a primary, but nobody renews its lease. After the
# TTL, pg-node2 is promoted: now TWO primaries accept writes, if clients can still
# reach pg-node1 directly. HAProxy stops routing to it (its REST API is dead), which
# is why clients should only ever connect through HAProxy.
```

```bash
# Expected: pg-node2 promotes (scenario 1 almost at once, scenario 2 after the TTL)
# If it doesn't:
journalctl -u patroni -n 50 --no-pager    # on pg-node2

# Check etcd leader key
etcdctl --endpoints=http://10.0.0.30:2379 get /service/pg-cluster/leader

# Force it:
patronictl -c /etc/patroni/patroni.yml failover pg-cluster --candidate pg-node2 --force
```

```bash
# Restore: bring pg-node1 back as replica. After scenario 2, stop the orphaned
# primary first (on pg-node1):
sudo -u postgres /usr/lib/postgresql/16/bin/pg_ctl stop -D /var/lib/postgresql/16/main -m fast
# then start Patroni, which rewinds it into a replica of the new leader
systemctl start patroni
sleep 15
patronictl -c /etc/patroni/patroni.yml list
```

**Write down**: Did auto-failover work? ___  
If not, what did the logs say? ___

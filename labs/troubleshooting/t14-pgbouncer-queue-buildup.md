# T-14 — PgBouncer Queue Buildup: Connections Waiting in Pool

**Frequency**: Common on traffic spikes
**Severity**: P1 — request latency explodes, clients timeout
**Related**: 2.4 PgBouncer

---

## The Symptom

Application latency spikes from 20ms to 5000ms. Monitoring shows PgBouncer has many connections in "waiting" state. PostgreSQL itself is fine — only a few connections active.

---

## Why This Happens

In transaction pooling mode, PgBouncer has `pool_size` server connections per database/user pair. If more clients arrive than pool_size, they queue. If the queue grows faster than transactions complete → latency explodes.

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

**Immediate**: Increase pool_size temporarily:
```ini
# pgbouncer.ini
pool_size = 50       # was 20
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

**Set client_login_timeout** so queued clients fail fast instead of piling up:
```ini
client_login_timeout = 10
query_timeout = 30
```

---

## Lab Exercise

```bash
# Requires PgBouncer installed and configured
# If not installed: apt install pgbouncer

# Basic pgbouncer.ini for this lab
cat > /tmp/pgbouncer_lab.ini << 'EOF'
[databases]
labdb = host=127.0.0.1 port=5432 dbname=postgres

[pgbouncer]
listen_port    = 6432
listen_addr    = 127.0.0.1
auth_type      = md5
auth_file      = /tmp/pgbouncer_users.txt
pool_mode      = transaction
pool_size      = 2              # tiny on purpose to demonstrate queuing
max_client_conn= 100
admin_users    = pgbouncer
EOF

echo '"pgbouncer" "pgbouncer"' > /tmp/pgbouncer_users.txt

pgbouncer -d /tmp/pgbouncer_lab.ini
```

```bash
# Simulate overload: 10 concurrent clients through a pool of 2
for i in $(seq 1 10); do
    psql -h 127.0.0.1 -p 6432 -U postgres labdb \
        -c "SELECT pg_sleep(2); SELECT $i AS id;" &
done

# Immediately check queue buildup
sleep 1
psql -h 127.0.0.1 -p 6432 -U pgbouncer pgbouncer -c "SHOW POOLS;"
# cl_waiting should be > 0

wait
```

```bash
# Increase pool_size and compare
# Edit ini: pool_size = 10
# Reload and repeat

# Clean up
pkill pgbouncer
```

**Write down**: With pool_size=2, max cl_waiting seen: ___  
With pool_size=10, max cl_waiting seen: ___

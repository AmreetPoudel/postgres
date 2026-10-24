# T-18 — OOM Killer Killed PostgreSQL

**Frequency**: Moderate on under-tuned servers
**Severity**: P0 — PostgreSQL crashes, needs restart
**Related**: 1.3 Memory Architecture, 2.1 postgresql.conf, 2.7 OS Tuning

---

## The Symptom

PostgreSQL crashes without warning. OS logs show:
```
kernel: Out of memory: Kill process 12345 (postgres) score 900 or sacrifice child
kernel: Killed process 12345 (postgres) total-vm:16GB, anon-rss:14GB
```

---

## Why This Happens

Total memory used by PostgreSQL > available RAM:
```
shared_buffers + (work_mem × active queries × sort/hash nodes) + per-connection overhead
  + maintenance_work_mem × autovacuum workers + OS overhead > RAM
```
(The OS page cache isn't part of the sum: the kernel can always reclaim it.)

Linux OOM killer picks the process using most memory and kills it.

It hardly matters *which* PostgreSQL process it picks. A backend killed with SIGKILL counts as a crash, so the postmaster terminates every other session and runs crash recovery: one killed backend is a full, if brief, outage. If the OOM killer picks the postmaster itself, PostgreSQL stays down until someone restarts it.

```
LOG:  server process (PID 12345) was terminated by signal 9: Killed
LOG:  terminating any other active server processes
```

---

## Diagnosis

```bash
# Check OOM events in kernel log
dmesg | grep -i "oom\|killed" | tail -20
grep -i "oom\|killed" /var/log/syslog | tail -20

# Current memory usage
free -h
cat /proc/meminfo | grep -E "MemTotal|MemAvailable|SwapTotal|SwapFree"

# PostgreSQL memory calculation
psql -c "SELECT
    pg_size_pretty(pg_size_bytes(current_setting('shared_buffers')))          AS shared_buf,
    pg_size_pretty(pg_size_bytes(current_setting('work_mem'))
        * current_setting('max_connections')::int * 3)                         AS worst_work_mem,
    current_setting('max_connections')                                          AS max_conn;"
```

---

## Fix

```
# postgresql.conf — reduce memory footprint
shared_buffers         = 8GB    # not more than 25% of RAM
work_mem               = 16MB   # keep low, set high per-session for analytics
max_connections        = 100    # reduce, use PgBouncer
```

**Stop the kernel from overcommitting** (2.7): with `vm.overcommit_memory = 2`, a query that asks for too much memory gets an `out of memory` error in its own session, instead of the OOM killer taking the whole server down. That's what the PostgreSQL documentation recommends for a dedicated server.

**If you keep overcommit on, at least protect the postmaster**, persistently, through systemd. The `PG_OOM_ADJUST_*` variables (described in the PostgreSQL docs on Linux memory overcommit) make each child process reset its own score, so a runaway backend is still the one that gets killed. RHEL's PGDG unit file already ships these lines:
```bash
systemctl edit postgresql@16-main      # RHEL/Rocky: systemctl edit postgresql-16
# then add:
#   [Service]
#   OOMScoreAdjust=-1000
#   Environment=PG_OOM_ADJUST_FILE=/proc/self/oom_score_adj
#   Environment=PG_OOM_ADJUST_VALUE=0
```

---

## Lab Exercise

```sql
-- Calculate your current theoretical max memory
SELECT
    current_setting('shared_buffers')                     AS shared_buffers,
    current_setting('work_mem')                           AS work_mem,
    current_setting('max_connections')                    AS max_connections,
    pg_size_pretty(
        pg_size_bytes(current_setting('work_mem')) *
        current_setting('max_connections')::bigint * 3
    )                                                     AS work_mem_worst_case;
```

```bash
# Check actual PostgreSQL memory usage. The postmaster's PID is the first line of
# postmaster.pid. (RSS includes the shared memory a process has touched, so adding
# up RSS across backends counts shared_buffers many times over.)
PG_PID=$(head -1 "$PGDATA/postmaster.pid")
grep -E "VmRSS|VmPeak|VmSize" /proc/$PG_PID/status
```

```bash
# Check OOM score
PG_PID=$(head -1 "$PGDATA/postmaster.pid")
cat /proc/$PG_PID/oom_score
# Lower score = less likely to be killed
# Default: ~500-900 for a large process

# Protect the postmaster, for this boot only (processes it forks from now on
# inherit the value, which is why the systemd setup above resets the children)
echo -100 | sudo tee /proc/$PG_PID/oom_score_adj
cat /proc/$PG_PID/oom_score   # should be lower now
```

**Write down**: Worst-case work_mem total: ___  
Does it exceed your server RAM? ___  
OOM score before adjustment: ___

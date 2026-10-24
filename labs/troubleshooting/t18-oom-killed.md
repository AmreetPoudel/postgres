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
shared_buffers + (work_mem × max_connections × sort_nodes) + OS page cache
  + OS overhead > Physical RAM
```

Linux OOM killer picks the process using most memory and kills it.

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

# OS — prevent OOM by reserving swap and setting limits
vm.swappiness          = 10
```

```bash
# Protect PostgreSQL from OOM killer (lower score = less likely to be killed)
PG_PID=$(pgrep -f "postgres: checkpointer")
echo -100 > /proc/$PG_PID/oom_score_adj
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
# Check actual PostgreSQL memory usage
PG_PID=$(pgrep -o -f "postgres: checkpointer" 2>/dev/null || echo "not found")
echo "Postmaster PID: $(pgrep -o postgres)"
cat /proc/$(pgrep -o postgres)/status | grep -E "VmRSS|VmPeak|VmSize"
```

```bash
# Check OOM score
PG_PID=$(pgrep -o postgres)
cat /proc/$PG_PID/oom_score
# Lower score = less likely to be killed
# Default: ~500-900 for a large process

# Protect PostgreSQL
echo -100 | sudo tee /proc/$PG_PID/oom_score_adj
cat /proc/$PG_PID/oom_score   # should be lower now
```

**Write down**: Worst-case work_mem total: ___  
Does it exceed your server RAM? ___  
OOM score before adjustment: ___

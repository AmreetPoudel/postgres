# Lab 002 — Memory Architecture & postgresql.conf Tuning

**Topics**: 1.3 Memory Architecture, 2.1 postgresql.conf
**Read first**: `curriculum/1-internals/1.3-memory-architecture.md`, `curriculum/2-config/2.1-postgresql-conf.md`
**Time**: ~60 minutes

---

## Goal

By the end of this lab you will have:
- Calculated the correct shared_buffers for your server
- Changed it and verified it took effect
- Observed a sort spilling to disk vs staying in memory
- Enabled slow query logging

---

## Task 1: Inventory Your Current Config vs Recommended

```sql
-- See all memory-related settings
SELECT name, setting, unit, context
FROM pg_settings
WHERE name IN (
    'shared_buffers', 'work_mem', 'maintenance_work_mem',
    'effective_cache_size', 'wal_buffers', 'max_connections'
)
ORDER BY name;
```

```bash
# How much RAM does this server have?
free -h
```

**Fill in:**

| Parameter | Current Value | Recommended (25%/75% rule) |
|---|---|---|
| shared_buffers | | 25% of RAM |
| effective_cache_size | | 75% of RAM |
| maintenance_work_mem | | 1–2 GB |
| work_mem | | 8–32 MB (conservative) |

---

## Task 2: Apply and Verify

```sql
-- Set values based on YOUR server's RAM
-- Example for 32 GB RAM:
ALTER SYSTEM SET shared_buffers       = '8GB';
ALTER SYSTEM SET effective_cache_size = '24GB';
ALTER SYSTEM SET maintenance_work_mem = '1GB';
ALTER SYSTEM SET work_mem             = '16MB';
```

```bash
# shared_buffers requires restart
pg_ctl restart -D "$PGDATA" -m fast

# Verify the change took effect
psql -c "SHOW shared_buffers;"
psql -c "SHOW effective_cache_size;"

# Check no pending_restart remains
psql -c "SELECT name, pending_restart FROM pg_settings WHERE pending_restart = true;"
```

---

## Task 3: Observe Sort Spill vs In-Memory

```sql
-- Create a table large enough to stress sorts
CREATE TABLE sort_lab AS
SELECT (random() * 10000000)::int AS val,
       md5(random()::text)        AS payload
FROM generate_series(1, 3000000);
```

```sql
-- Force a sort with very low work_mem
SET work_mem = '1MB';

EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT val, payload FROM sort_lab ORDER BY val;

-- What does "Sort Method" say?
-- Expect: Sort Method: external merge  Disk: xxxkB
```

```sql
-- Now with adequate work_mem
SET work_mem = '512MB';

EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT val, payload FROM sort_lab ORDER BY val;

-- What does "Sort Method" say now?
-- Expect: Sort Method: quicksort  Memory: xxxkB

RESET work_mem;
```

**Write down:**
- At 1 MB work_mem, sort method was: _____________
- At 512 MB work_mem, sort method was: _____________
- Latency difference: _____ ms vs _____ ms

---

## Task 4: The Connection × work_mem Math

```sql
-- Worst-case memory calculation
SELECT
    current_setting('work_mem')                               AS work_mem,
    current_setting('max_connections')::int                   AS max_conn,
    pg_size_pretty(
        pg_size_bytes(current_setting('work_mem')) *
        current_setting('max_connections')::bigint * 3
    )                                                         AS worst_case_ram;
-- 3 = assume 3 sort/hash nodes per query (conservative)
```

**Write down:** worst_case_ram = _____. Does this exceed your server's RAM? ______

If YES → work_mem is set too high and you risk OOM. The fix is to use per-session SET for heavy analytical queries rather than raising the global default.

---

## Task 5: Enable Slow Query Logging

```sql
ALTER SYSTEM SET log_min_duration_statement = 500;   -- log queries >500ms
ALTER SYSTEM SET log_checkpoints             = on;
ALTER SYSTEM SET log_lock_waits              = on;
ALTER SYSTEM SET log_temp_files              = 0;    -- log all temp file usage
SELECT pg_reload_conf();                             -- no restart needed
```

```bash
# Trigger a slow query
psql -c "SELECT pg_sleep(1);"

# Find it in logs
grep "duration:" "$PGDATA"/log/postgresql-*.log | tail -5
```

---

## Cleanup

```sql
DROP TABLE sort_lab;
```

---

## Completion Checklist

- [ ] Calculated shared_buffers correctly for this server's RAM
- [ ] Applied changes and restarted PostgreSQL
- [ ] Verified settings took effect with SHOW
- [ ] Observed sort spill to disk vs memory with EXPLAIN ANALYZE
- [ ] Understood the work_mem × connections × 3 worst-case calculation
- [ ] Enabled slow query logging and verified a slow query appears in logs
- [ ] Updated `progress.md`

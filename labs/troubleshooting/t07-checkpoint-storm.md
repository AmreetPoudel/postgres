# T-07 — Checkpoint Storm: Periodic I/O Spikes

**Frequency in production**: Common on write-heavy systems with defaults
**Severity**: P2 — periodic latency spikes every 5 minutes
**Related**: 2.6 Checkpoint and WAL Config

---

## The Symptom

Latency graph shows regular spikes every 5 minutes. System is slow for 10-30 seconds then fine. I/O utilization hits 100% in short bursts.

---

## Why This Happens

Default `checkpoint_timeout = 5min`. Every 5 minutes, all dirty pages flush at once = I/O burst.

---

## Diagnosis

```sql
SELECT checkpoints_timed,
       checkpoints_req,
       round(checkpoints_req::numeric/nullif(checkpoints_timed+checkpoints_req,0)*100,1)
           AS pct_forced,
       checkpoint_write_time/1000.0 AS write_sec,
       buffers_checkpoint,
       buffers_backend         -- nonzero = backends flushing own pages (bad)
FROM pg_stat_bgwriter;

-- checkpoints_req >> checkpoints_timed → increase max_wal_size
-- buffers_backend > 0 → checkpoints can't keep up with write rate
```

```bash
grep "checkpoint" "$PGDATA"/log/postgresql-*.log | tail -10
```

---

## Fix

```
# postgresql.conf
checkpoint_timeout           = 30min
max_wal_size                 = 4GB
checkpoint_completion_target = 0.9
log_checkpoints              = on
```

---

## Lab Exercise

```sql
-- Enable checkpoint logging
ALTER SYSTEM SET log_checkpoints = on;
ALTER SYSTEM SET checkpoint_timeout = '1min';   -- short for demo
ALTER SYSTEM SET max_wal_size = '64MB';         -- tiny to force frequent checkpoints
SELECT pg_reload_conf();

-- Reset stats
SELECT pg_stat_reset_shared('bgwriter');

-- Generate WAL to trigger checkpoints
DO $$
BEGIN
    FOR i IN 1..20 LOOP
        EXECUTE 'CREATE TABLE IF NOT EXISTS chk_test_' || i || ' AS
            SELECT md5(j::text) FROM generate_series(1,5000) j';
    END LOOP;
END;
$$;

-- Check checkpoint stats
SELECT checkpoints_timed, checkpoints_req,
       round(checkpoints_req::numeric/nullif(checkpoints_timed+checkpoints_req,0)*100,1) AS pct_forced
FROM pg_stat_bgwriter;

-- View checkpoint log entries
-- grep "checkpoint" "$PGDATA"/log/postgresql-*.log | tail -10
```

```sql
-- Restore sane settings
ALTER SYSTEM SET checkpoint_timeout = '30min';
ALTER SYSTEM SET max_wal_size = '4GB';
SELECT pg_reload_conf();

-- Cleanup
DO $$ BEGIN
    FOR i IN 1..20 LOOP
        EXECUTE 'DROP TABLE IF EXISTS chk_test_' || i;
    END LOOP;
END; $$;
```

**Write down**: How many checkpoints_req vs checkpoints_timed? ___  
What does pct_forced tell you? ___

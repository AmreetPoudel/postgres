# T-07 — Checkpoint Storm: Periodic I/O Spikes

**Frequency in production**: Common on write-heavy systems with defaults
**Severity**: P2 — periodic latency spikes every 5 minutes
**Related**: 2.6 Checkpoint and WAL Config

---

## The Symptom

Latency graph shows regular spikes every 5 minutes. System is slow for 10-30 seconds then fine. I/O utilization hits 100% in short bursts.

---

## Why This Happens

Default `checkpoint_timeout = 5min`, and on a busy server the default `max_wal_size = 1GB` forces checkpoints even more often. The writes themselves are spread out (`checkpoint_completion_target = 0.9`), but every checkpoint still causes two bursts: the fsync of everything written, at the end, and a flood of full-page images in WAL right after it starts (the first change to every page after a checkpoint logs the whole 8KB page).

---

## Diagnosis

```sql
SELECT checkpoints_timed,
       checkpoints_req,
       round(checkpoints_req::numeric/nullif(checkpoints_timed+checkpoints_req,0)*100,1)
           AS pct_forced,
       checkpoint_write_time/1000.0 AS write_sec,
       buffers_checkpoint,
       buffers_backend         -- a large share = backends flushing their own pages
FROM pg_stat_bgwriter;

-- checkpoints_req >> checkpoints_timed → increase max_wal_size
-- buffers_backend a large share of writes → bgwriter can't keep clean buffers ready

-- Full-page images: lots of them = checkpoints are frequent (PG14+)
SELECT wal_records, wal_fpi FROM pg_stat_wal;
```

```bash
grep "checkpoint" "$PGDATA"/log/postgresql-*.log | tail -10
# Look for "checkpoints are occurring too frequently" and long sync= times
# (Debian/Ubuntu: /var/log/postgresql/postgresql-16-main.log)
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

-- Generate ~250MB of WAL (about 4× max_wal_size) to force requested checkpoints.
-- (Twenty 5,000-row tables would write only ~9MB: not enough to trigger anything.)
CREATE TABLE chk_test AS
SELECT md5(j::text) AS v FROM generate_series(1, 3000000) j;

-- Check checkpoint stats
SELECT checkpoints_timed, checkpoints_req,
       round(checkpoints_req::numeric/nullif(checkpoints_timed+checkpoints_req,0)*100,1) AS pct_forced
FROM pg_stat_bgwriter;

-- View checkpoint log entries: you should see "checkpoints are occurring too frequently"
-- grep "checkpoint" "$PGDATA"/log/postgresql-*.log | tail -10
```

```sql
-- Put back whatever you had before the lab
ALTER SYSTEM RESET checkpoint_timeout;
ALTER SYSTEM RESET max_wal_size;
SELECT pg_reload_conf();

-- Cleanup
DROP TABLE chk_test;
```

**Write down**: How many checkpoints_req vs checkpoints_timed? ___  
What does pct_forced tell you? ___

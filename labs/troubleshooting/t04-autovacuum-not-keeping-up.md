# T-04 — Autovacuum Not Keeping Up (Bloat Emergency)

**Frequency in production**: Common on high-write tables
**Severity**: P2 → P1 if wraparound approached
**Related**: 1.7 VACUUM, 2.5 Autovacuum Tuning

---

## The Symptom

Table queries get slower over weeks. `n_dead_tup` grows continuously. `last_autovacuum` is hours or days old on a busy table.

---

## Why This Happens

1. **Scale factor too large** — default 20% means 10M dead tuples on a 50M-row table before vacuum triggers
2. **Long transactions blocking vacuum** — vacuum cannot clean tuples visible to old transactions
3. **Cost throttle too aggressive** — vacuum sleeps 20ms between chunks on fast NVMe (crawling)
4. **Not enough workers** — all 3 workers busy with other tables

---

## Diagnosis

```sql
-- Tables with worst bloat
SELECT relname, n_live_tup, n_dead_tup,
       round(n_dead_tup::numeric/nullif(n_live_tup+n_dead_tup,0)*100,1) AS dead_pct,
       last_autovacuum,
       now() - last_autovacuum AS since_vacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC LIMIT 10;

-- Is autovacuum running right now?
SELECT pid, now() - query_start AS duration, query
FROM pg_stat_activity WHERE backend_type = 'autovacuum worker';

-- Oldest transaction (blocking vacuum)
SELECT pid, now() - xact_start AS age, state, left(query,80)
FROM pg_stat_activity
WHERE xact_start IS NOT NULL ORDER BY age DESC LIMIT 5;

-- XID wraparound risk
SELECT datname, age(datfrozenxid) AS xid_age
FROM pg_database ORDER BY xid_age DESC;
-- Alert: > 500M. Emergency: > 1.5B
```

---

## Immediate Fix

```sql
-- Manual vacuum NOW on the worst table
VACUUM (VERBOSE, ANALYZE) orders;
```

```bash
# If multiple tables need attention
vacuumdb -d mydb --analyze --jobs 4 --verbose
```

---

## Root Cause Fix

```sql
-- Per-table aggressive settings for hot tables
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor  = 0.01,
    autovacuum_vacuum_threshold     = 5000,
    autovacuum_analyze_scale_factor = 0.005,
    autovacuum_vacuum_cost_delay    = 0,
    autovacuum_vacuum_cost_limit    = 1000
);
```

```
# postgresql.conf
autovacuum_max_workers          = 6
autovacuum_vacuum_cost_limit    = 800
autovacuum_naptime              = 30s
log_autovacuum_min_duration     = 250
```

---

## Lab Exercise

```sql
-- Create a bloat-prone table with default autovacuum
CREATE TABLE bloat_watch (
    id      serial PRIMARY KEY,
    counter int  DEFAULT 0,
    data    text DEFAULT md5(random()::text)
) WITH (autovacuum_vacuum_scale_factor = 0.2);

INSERT INTO bloat_watch (counter) SELECT 0 FROM generate_series(1, 500000);

-- Generate dead tuples
UPDATE bloat_watch SET counter = counter + 1;

-- Check: autovacuum threshold
SELECT 50 + (0.2 * 500000) AS trigger_threshold;  -- 100,050
-- We have 500,000 dead tuples > 100,050 → autovacuum should trigger

-- Monitor (check every 30 seconds for 5 minutes)
SELECT relname, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables WHERE relname = 'bloat_watch';

-- Enable logging to confirm
ALTER SYSTEM SET log_autovacuum_min_duration = 0;
SELECT pg_reload_conf();

-- Now switch to aggressive settings and compare speed
ALTER TABLE bloat_watch SET (
    autovacuum_vacuum_scale_factor = 0.01,
    autovacuum_vacuum_cost_delay   = 0
);
UPDATE bloat_watch SET counter = counter + 1;   -- create more dead tuples
-- Should autovacuum faster now

-- Clean up
DROP TABLE bloat_watch;
ALTER SYSTEM RESET log_autovacuum_min_duration;
SELECT pg_reload_conf();
```

**Write down**: With default settings, autovacuum fired after __ minutes  
With aggressive settings, autovacuum fired after __ minutes

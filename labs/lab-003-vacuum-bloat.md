# Lab 003 — VACUUM, Bloat, and Dead Tuples

**Topics**: 1.7 VACUUM, 2.5 Autovacuum Tuning
**Read first**: `curriculum/1-internals/1.7-vacuum.md`, `curriculum/2-config/2.5-autovacuum-tuning.md`
**Time**: ~60 minutes

---

## Goal

By the end of this lab you will have:
- Created measurable table bloat and observed it with pg_stat_user_tables
- Watched VACUUM reclaim dead tuples
- Proved that regular VACUUM does NOT shrink the file
- Watched VACUUM FULL actually shrink the file
- Set aggressive per-table autovacuum settings
- Checked transaction ID age on all databases

---

## Task 1: Create Bloat and Measure It

```sql
-- Disable autovacuum so it doesn't clean up before we can measure
CREATE TABLE bloat_lab (
    id      serial PRIMARY KEY,
    counter int    DEFAULT 0,
    data    text   DEFAULT md5(random()::text)
) WITH (autovacuum_enabled = false);

-- Insert 500,000 rows
INSERT INTO bloat_lab (counter)
SELECT 0 FROM generate_series(1, 500000);

-- Baseline measurement
SELECT relname, n_live_tup, n_dead_tup,
       pg_size_pretty(pg_relation_size('bloat_lab')) AS size
FROM pg_stat_user_tables
WHERE relname = 'bloat_lab';
```

Record: baseline size = _____

```sql
-- UPDATE all 500,000 rows (creates 500,000 dead versions)
UPDATE bloat_lab SET counter = counter + 1;

-- Measure bloat BEFORE vacuum
SELECT relname, n_live_tup, n_dead_tup,
       pg_size_pretty(pg_relation_size('bloat_lab')) AS size
FROM pg_stat_user_tables
WHERE relname = 'bloat_lab';
```

Record: size after UPDATE = _____ | dead tuples = _____

**Question**: Why is the table bigger even though the row count didn't change?

---

## Task 2: Run VACUUM and Observe

```sql
-- Verbose VACUUM so you see exactly what it does
VACUUM (VERBOSE, ANALYZE) bloat_lab;
```

Read the verbose output carefully. Find and write down:
- Rows removed (dead tuples cleaned): _____
- Pages scanned: _____

```sql
-- Measure AFTER vacuum
SELECT relname, n_live_tup, n_dead_tup,
       pg_size_pretty(pg_relation_size('bloat_lab')) AS size
FROM pg_stat_user_tables
WHERE relname = 'bloat_lab';
```

Record: size after VACUUM = _____  |  dead tuples remaining = _____

**Key question**: Did the size shrink? Why or why not?

---

## Task 3: VACUUM FULL — The File Actually Shrinks

```sql
-- Create another round of bloat
UPDATE bloat_lab SET counter = counter + 1;

SELECT pg_size_pretty(pg_relation_size('bloat_lab')) AS bloated_size;
```

Now run VACUUM FULL. On a table this size it finishes in about a second, too fast to race by hand, so make the lock visible with three terminals:

**Terminal 1 (hold a normal read lock, like a long report would):**
```sql
BEGIN;
SELECT count(*) FROM bloat_lab;
-- don't COMMIT yet
```

**Terminal 2:**
```sql
VACUUM FULL VERBOSE bloat_lab;
-- Waits: it needs ACCESS EXCLUSIVE, which conflicts with Terminal 1's ACCESS SHARE
```

**Terminal 3:**
```sql
SELECT count(*) FROM bloat_lab;
-- Also waits, even though Terminal 1's identical query ran fine.
-- It queues behind Terminal 2's pending ACCESS EXCLUSIVE request.
```

**Terminal 4 (optional): see the queue**
```sql
SELECT pid, pg_blocking_pids(pid) AS blocked_by, wait_event_type, left(query, 40) AS query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

Now `COMMIT;` in Terminal 1. Terminal 2's VACUUM FULL runs, and Terminal 3's SELECT completes right after it. That's the ACCESS EXCLUSIVE lock, plus the lock queue behind it: T-02 is this same pile-up in production.

After VACUUM FULL completes:

```sql
SELECT pg_size_pretty(pg_relation_size('bloat_lab')) AS post_full_vacuum_size;
```

Record: size after VACUUM FULL = _____

Did the file shrink this time? _____ How much did it shrink? _____

**Takeaway**: VACUUM FULL = file shrinks + ACCESS EXCLUSIVE lock. Regular VACUUM = dead tuples reclaimed for reuse + no lock on reads.

---

## Task 4: Autovacuum Threshold Math

```sql
-- What is the autovacuum trigger threshold for a 500,000 row table?
SELECT
    current_setting('autovacuum_vacuum_threshold')::numeric AS threshold,
    current_setting('autovacuum_vacuum_scale_factor')::numeric AS scale_factor,
    500000 AS live_rows,
    current_setting('autovacuum_vacuum_threshold')::numeric +
        (current_setting('autovacuum_vacuum_scale_factor')::numeric * 500000)
        AS trigger_at_n_dead_tuples;
```

Record: autovacuum triggers on this table when dead tuples exceed = _____

Now set aggressive per-table settings:

```sql
-- Re-enable autovacuum and set aggressive thresholds
ALTER TABLE bloat_lab SET (
    autovacuum_enabled             = true,
    autovacuum_vacuum_scale_factor = 0.01,   -- 1% instead of 20%
    autovacuum_vacuum_threshold    = 1000,
    autovacuum_vacuum_cost_delay   = 0,
    autovacuum_vacuum_cost_limit   = 800
);

-- New trigger threshold with aggressive settings:
SELECT 1000 + (0.01 * 500000) AS new_trigger_threshold;
-- Should be 6,000 instead of the previous large number
```

Generate enough dead tuples to trigger autovacuum:

```sql
UPDATE bloat_lab SET counter = counter + 1 WHERE id <= 7000;
-- 7,000 dead tuples > 6,000 threshold → autovacuum should run
```

Watch for autovacuum worker in a loop (run in bash):

```bash
for i in $(seq 1 24); do
    sleep 5
    psql -c "SELECT pid, now() - query_start AS age, query
             FROM pg_stat_activity
             WHERE backend_type = 'autovacuum worker';"
done
```

Did autovacuum run? Check:

```sql
SELECT relname, last_autovacuum, n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'bloat_lab';
```

---

## Task 5: Transaction ID Wraparound Check

```sql
-- Check XID age on all databases (should be << 1 billion)
SELECT datname,
       age(datfrozenxid)                       AS xid_age,
       round(age(datfrozenxid) / 2000000000.0 * 100, 1) AS pct_to_wraparound
FROM pg_database
ORDER BY age(datfrozenxid) DESC;

-- Per-table freeze age (find oldest unfrozen tables)
SELECT relname, age(relfrozenxid) AS xid_age
FROM pg_class
WHERE relkind = 'r' AND relfrozenxid != 0
ORDER BY age(relfrozenxid) DESC
LIMIT 10;
```

Record: oldest database XID age = _____  |  % to wraparound = _____%

Alert threshold: > 500 million. Critical: > 1 billion (VACUUM's failsafe engages at 1.6 billion).

---

## Cleanup

```sql
DROP TABLE bloat_lab;
```

---

## Completion Checklist

- [ ] Created 500,000 dead tuples and measured the bloat
- [ ] Ran VACUUM VERBOSE and read the output
- [ ] Confirmed regular VACUUM does NOT shrink the file
- [ ] Ran VACUUM FULL and confirmed the file DOES shrink
- [ ] Proved VACUUM FULL holds ACCESS EXCLUSIVE lock
- [ ] Calculated autovacuum trigger threshold (default and tuned)
- [ ] Set aggressive per-table autovacuum settings
- [ ] Verified autovacuum ran with the aggressive settings
- [ ] Checked XID wraparound age on all databases
- [ ] Updated `progress.md`

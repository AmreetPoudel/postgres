# T-05 — Stale Statistics: Sudden Query Slowdown After Data Change

**Frequency in production**: Very common
**Severity**: P2 — specific queries 10-1000x slower
**Related**: 1.8 Query Lifecycle

---

## The Symptom

A query that ran in 50ms for months suddenly takes 30 seconds. Nothing changed in the code. Happened right after a large data import or table growth.

---

## Why This Happens

The planner uses `pg_statistic` row count estimates. If stats are stale, it picks the wrong plan.

Classic: table grew from 1K to 50M rows after a data load. Statistics still say "100 rows match this filter". Planner picks index scan (good for 100 rows, terrible for 5M rows).

---

## Diagnosis

```sql
-- 1. Get estimated vs actual rows
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE status = 'pending';
-- "rows=100" (estimate) vs "actual rows=5000000" → stale stats

-- 2. When were stats last updated?
SELECT relname, last_analyze, last_autoanalyze, n_mod_since_analyze
FROM pg_stat_user_tables WHERE relname = 'orders';
-- n_mod_since_analyze: rows changed since last analyze

-- 3. What does planner think about the column?
SELECT most_common_vals, most_common_freqs, n_distinct
FROM pg_stats WHERE tablename='orders' AND attname='status';
```

---

## Immediate Fix

```sql
ANALYZE orders;
-- Then re-check EXPLAIN — plan should change
```

---

## If ANALYZE Doesn't Fix It

```sql
-- Increase statistics depth for this column
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;
ANALYZE orders;
```

---

## Lab Exercise

```sql
CREATE TABLE stale_stats_lab (
    id       serial PRIMARY KEY,
    category text,
    amount   numeric(10,2)
);
CREATE INDEX ON stale_stats_lab(category);

-- Small initial load
INSERT INTO stale_stats_lab (category, amount)
SELECT 'cat_' || (i%5), (random()*100)::numeric(10,2)
FROM generate_series(1, 1000) i;

ANALYZE stale_stats_lab;

-- See initial plan
EXPLAIN (ANALYZE) SELECT * FROM stale_stats_lab WHERE category = 'cat_0';
-- Note estimated rows: ___
```

```sql
-- Massive data load without ANALYZE
INSERT INTO stale_stats_lab (category, amount)
SELECT 'cat_' || (i%5), (random()*100)::numeric(10,2)
FROM generate_series(1, 1000000) i;

-- Check plan with stale stats
EXPLAIN (ANALYZE) SELECT * FROM stale_stats_lab WHERE category = 'cat_0';
-- Estimated rows: ___  Actual rows: ___
-- Is the plan still correct?
```

```sql
-- Fix
ANALYZE stale_stats_lab;

-- Check plan again
EXPLAIN (ANALYZE) SELECT * FROM stale_stats_lab WHERE category = 'cat_0';
-- Estimated rows should now be close to actual

DROP TABLE stale_stats_lab;
```

**Write down**: Before ANALYZE: estimate=___ actual=___  
After ANALYZE: estimate=___ actual=___

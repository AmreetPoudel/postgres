# T-17 — Query Plan Regression After PostgreSQL Upgrade

**Frequency**: Moderate — happens after major version upgrades
**Severity**: P2 — specific queries regress, rest fine
**Related**: 1.8 Query Lifecycle

---

## The Symptom

After upgrading from PostgreSQL 14 to 16, a specific query that ran in 100ms now takes 10 seconds. The data hasn't changed. The query hasn't changed. Only the version changed.

---

## Why This Happens

The planner uses statistics + cost model constants. After a major-version upgrade:
- **Statistics are missing.** `pg_upgrade` doesn't carry optimizer statistics over (until PostgreSQL 18), so the new cluster plans with default guesses until ANALYZE runs. This is the #1 cause. Run `vacuumdb --all --analyze-in-stages` right after the upgrade: it produces rough statistics within seconds and full ones afterwards.
- **Settings didn't come along.** A new cluster starts with the default `random_page_cost`, `effective_cache_size`, `work_mem`... unless you carried your config over.
- **The planner itself changed.** New strategies (incremental sort in 13, memoize in 14, and so on) occasionally pick a worse plan for a particular data distribution.

---

## Diagnosis

```sql
-- Get current plan
EXPLAIN (ANALYZE, BUFFERS) <slow_query>;

-- Compare with an alternative join method:
SET enable_hashjoin = off;
EXPLAIN (ANALYZE, BUFFERS) <slow_query>;
RESET enable_hashjoin;

-- Try forcing the old plan type:
SET enable_nestloop = off;
EXPLAIN (ANALYZE, BUFFERS) <slow_query>;
RESET enable_nestloop;

-- Update statistics (always first step)
ANALYZE <table>;
EXPLAIN (ANALYZE, BUFFERS) <slow_query>;
```

---

## Fix Options

```sql
-- 1. ANALYZE first (most common fix)
ANALYZE table_name;

-- 2. Increase statistics for key column
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;
ANALYZE orders;

-- 3. Hint with enable_* (per session or via pg_hint_plan extension)
SET enable_hashjoin = off;   -- steer away from hash joins (nested loop or merge join instead)

-- 4. Use pg_hint_plan extension for targeted hints
-- /*+ HashJoin(a b) */ SELECT ...

-- 5. Pin hints without changing the query text: pg_hint_plan's hint table (hint_plan.hints)
```

---

## Lab Exercise

Simulate the #1 cause, a freshly upgraded cluster without statistics, and watch the plan change.

```sql
-- A skewed table: 99% 'common', 1% 'rare'
CREATE TABLE plan_reg_lab (id int, category text, amount numeric)
    WITH (autovacuum_enabled = false);   -- so autoanalyze can't interfere mid-lab
INSERT INTO plan_reg_lab
SELECT i, CASE WHEN i % 100 = 0 THEN 'rare' ELSE 'common' END, random() * 10000
FROM generate_series(1, 1000000) i;
CREATE INDEX ON plan_reg_lab(category);
ANALYZE plan_reg_lab;

-- Baseline, with statistics
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*), sum(amount) FROM plan_reg_lab WHERE category = 'common';
-- Plan type: ___   estimated rows: ___   time: ___
```

```sql
-- What a freshly pg_upgraded cluster looks like: no statistics at all.
-- (Superuser only, and only on a lab table: never do this in production.)
DELETE FROM pg_statistic WHERE starelid = 'plan_reg_lab'::regclass;

EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*), sum(amount) FROM plan_reg_lab WHERE category = 'common';
-- Plan type: ___   estimated rows: ___ (actual: about 990,000)   time: ___
```

```sql
-- The fix after every major upgrade: statistics
ANALYZE plan_reg_lab;      -- after a real upgrade: vacuumdb --all --analyze-in-stages

EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*), sum(amount) FROM plan_reg_lab WHERE category = 'common';
-- Back to the baseline plan? ___

DROP TABLE plan_reg_lab;
```

**Write down**: Estimated rows with and without statistics: ___ vs ___. Which plan did the planner pick without them, and why does a default guess lead there?

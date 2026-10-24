# T-17 — Query Plan Regression After PostgreSQL Upgrade

**Frequency**: Moderate — happens after major version upgrades
**Severity**: P2 — specific queries regress, rest fine
**Related**: 1.8 Query Lifecycle

---

## The Symptom

After upgrading from PostgreSQL 14 to 16, a specific query that ran in 100ms now takes 10 seconds. The data hasn't changed. The query hasn't changed. Only the version changed.

---

## Why This Happens

The planner uses statistics + cost model constants. Between versions:
- Cost constants may change (seq_page_cost, cpu_tuple_cost)
- Planner logic improvements sometimes pick worse plans for specific data distributions
- Statistics format may change

---

## Diagnosis

```sql
-- Get current plan
EXPLAIN (ANALYZE, BUFFERS) <slow_query>;

-- Compare with what an explicit join order would give:
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
SET enable_hashjoin = off;   -- force nested loop

-- 4. Use pg_hint_plan extension for targeted hints
-- /*+ HashJoin(a b) */ SELECT ...

-- 5. Lock a stable plan with pg_plan_cache (advanced)
```

---

## Lab Exercise

```sql
-- Simulate a bad plan by manipulating cost parameters
CREATE TABLE plan_reg_lab AS
SELECT i AS id,
       (i % 100)::text AS category,
       (random()*10000)::numeric AS amount
FROM generate_series(1, 500000) i;

CREATE INDEX ON plan_reg_lab(category);
CREATE INDEX ON plan_reg_lab(id);
ANALYZE plan_reg_lab;

-- Baseline plan
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM plan_reg_lab WHERE category = '42' ORDER BY id LIMIT 10;
-- Note plan type and cost: ___
```

```sql
-- Simulate "planner regression" by adjusting cost parameters
SET random_page_cost = 100;   -- makes index look very expensive
SET cpu_tuple_cost   = 0.5;   -- makes sequential work look cheap

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM plan_reg_lab WHERE category = '42' ORDER BY id LIMIT 10;
-- Plan changes: ___
```

```sql
-- Fix: reset to correct values for your hardware
RESET random_page_cost;
RESET cpu_tuple_cost;

-- For NVMe: set appropriate values
SET random_page_cost = 1.1;
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM plan_reg_lab WHERE category = '42' ORDER BY id LIMIT 10;
-- Back to good plan: ___

DROP TABLE plan_reg_lab;
```

**Write down**: How did changing random_page_cost affect the plan choice? ___

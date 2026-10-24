# T-25 — Full Slow Query Investigation Workflow

**Frequency**: Daily in production
**Severity**: Varies
**Related**: 1.8 Query Lifecycle, 2.8 Monitoring

---

## The Scenario

You get an alert: "p95 API latency is 4 seconds, up from 50ms baseline."  
You need to find the cause in PostgreSQL in under 10 minutes.

---

## The 10-Minute Investigation Protocol

### Minute 1-2: Is it PostgreSQL or something else?

```bash
# Database server: CPU, I/O, memory OK?
top
iostat -x 1 3
free -h

# PostgreSQL is responding?
psql -c "SELECT 1;" && echo "PostgreSQL is up"
```

### Minute 2-4: Active queries right now

```sql
-- What is running and for how long?
SELECT pid,
       now() - query_start AS duration,
       wait_event_type,
       wait_event,
       state,
       left(query, 100) AS query
FROM pg_stat_activity
WHERE state != 'idle'
  AND pid != pg_backend_pid()
ORDER BY duration DESC LIMIT 20;

-- Any lock waits?
SELECT count(*) FROM pg_stat_activity WHERE wait_event_type = 'Lock';
```

### Minute 4-6: Which query is the culprit?

```sql
-- Top queries by total time since last reset
SELECT
    round(total_exec_time::numeric) AS total_ms,
    calls,
    round(mean_exec_time::numeric)  AS avg_ms,
    round(max_exec_time::numeric)   AS max_ms,
    rows / calls                    AS rows_per_call,
    left(query, 120)                AS query
FROM pg_stat_statements
ORDER BY total_exec_time DESC LIMIT 10;

-- Or top by MEAN time (find the slowest individual calls)
SELECT round(mean_exec_time::numeric) AS avg_ms,
       max_exec_time,
       calls,
       left(query, 120)
FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 10;
```

### Minute 6-8: Get the plan for the slow query

```sql
-- Get the plan (fill in the actual slow query)
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
<slow query here>;

-- Red flags in plan output:
-- "Seq Scan on large_table"          → missing index
-- "Sort Method: external merge Disk" → work_mem too low
-- "Rows Removed by Filter: 9999999"  → bad filter, needs index
-- Estimated rows << Actual rows      → stale stats
-- "Hash Join" with huge Memory Usage → high work_mem consumption
```

### Minute 8-10: Fix

```sql
-- Fix A: Stale statistics
ANALYZE table_name;
EXPLAIN (ANALYZE, BUFFERS) <slow query>;

-- Fix B: Missing index
CREATE INDEX CONCURRENTLY idx_name ON table_name(column_name);

-- Fix C: work_mem too low (fix for THIS session)
SET work_mem = '256MB';
-- (Investigate if a global increase is warranted)

-- Fix D: Blocking transaction. Find the ROOT blocker first. (Killing the sessions
-- with wait_event_type = 'Lock' only hurts the victims, not the cause.)
SELECT pid, state, now() - xact_start AS xact_age, left(query, 60) AS query
FROM pg_stat_activity
WHERE pid IN (SELECT unnest(pg_blocking_pids(pid)) FROM pg_stat_activity)
  AND cardinality(pg_blocking_pids(pid)) = 0;
-- then, once you know what it is:
SELECT pg_terminate_backend(12345);   -- the root blocker's pid
```

---

## Lab Exercise: Full Investigation

```sql
-- Setup: a slow scenario for you to diagnose
CREATE TABLE investigation_lab AS
SELECT
    i                               AS id,
    (i % 10000)                     AS customer_id,
    (random() * 10000)::numeric(10,2) AS amount,
    now() - ((random()*365)::int || ' days')::interval AS created_at
FROM generate_series(1, 2000000) i;

-- Purposely don't create index or analyze
```

```sql
-- Simulate the "slow API query"
\timing on
SELECT customer_id, sum(amount) AS total
FROM investigation_lab
WHERE customer_id = 4242
GROUP BY customer_id;
\timing off
```

```sql
-- Step 1: Get the plan
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, sum(amount) FROM investigation_lab
WHERE customer_id = 4242 GROUP BY customer_id;
-- What is the plan type? Seq Scan or Index Scan? ___
-- Estimated rows: ___  Actual rows: ___
```

```sql
-- Step 2: Fix stale stats
ANALYZE investigation_lab;
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, sum(amount) FROM investigation_lab
WHERE customer_id = 4242 GROUP BY customer_id;
-- Did the plan change? ___
-- Estimated rows now: ___
```

```sql
-- Step 3: Create index
CREATE INDEX CONCURRENTLY idx_inv_customer ON investigation_lab(customer_id);

EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, sum(amount) FROM investigation_lab
WHERE customer_id = 4242 GROUP BY customer_id;
-- Plan type now: ___
```

```sql
-- Step 4: Benchmark improvement
\timing on
SELECT customer_id, sum(amount) FROM investigation_lab
WHERE customer_id = 4242 GROUP BY customer_id;
\timing off

DROP TABLE investigation_lab;
```

**Write down**:
- Initial query time: ___ ms
- After ANALYZE: ___ ms  
- After index: ___ ms
- Total speedup: ___×
- What was the root cause? ___

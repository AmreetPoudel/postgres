# T-15 — Missing Index: Sequential Scan on Large Table

**Frequency**: Very common
**Severity**: P2 — specific query is slow, rest of system fine
**Related**: 1.9 Index Internals, 1.8 Query Lifecycle

---

## The Symptom

One specific query runs in 30 seconds. `pg_stat_statements` shows it as the top consumer. EXPLAIN shows "Seq Scan" on a 50M-row table.

---

## Diagnosis

```sql
-- Find the slow query
SELECT left(query,100), calls, mean_exec_time, total_exec_time
FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 10;

-- Get the plan
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 42 AND status = 'pending';

-- Look for:
-- Seq Scan on orders (cost=... rows=50000000)
-- Buffers: shared read=xxxxxxx  ← reading millions of pages from disk
```

---

## Fix

```sql
-- Add index
CREATE INDEX CONCURRENTLY idx_orders_customer_status
    ON orders(customer_id, status);

-- Recheck plan
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 42 AND status = 'pending';
-- Should now show: Index Scan using idx_orders_customer_status
```

---

## Lab Exercise

```sql
-- Create a large table without index
CREATE TABLE large_orders (
    id          bigserial PRIMARY KEY,
    customer_id int,
    status      text,
    amount      numeric(10,2),
    created_at  timestamp DEFAULT now()
);

INSERT INTO large_orders(customer_id, status, amount)
SELECT
    (random()*100000)::int,
    CASE WHEN random()<0.05 THEN 'pending' ELSE 'completed' END,
    (random()*1000)::numeric(10,2)
FROM generate_series(1, 1000000);

ANALYZE large_orders;
```

```sql
-- Run without index, observe plan and timing
\timing on
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM large_orders
WHERE customer_id = 42 AND status = 'pending';
-- Note: Seq Scan, timing: ___
```

```sql
-- Add index
CREATE INDEX CONCURRENTLY idx_lo_cust_status
    ON large_orders(customer_id, status);

-- Run again
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM large_orders
WHERE customer_id = 42 AND status = 'pending';
-- Note: Index Scan, timing: ___
```

```sql
-- Check which indexes exist and if they're being used
SELECT relname, indexrelname, idx_scan
FROM pg_stat_user_indexes WHERE relname = 'large_orders';

DROP TABLE large_orders;
\timing off
```

**Write down**: Query time without index: ___ ms  
Query time with index: ___ ms  
Speedup factor: ___×

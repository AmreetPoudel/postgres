# T-10 — Deadlock: Application Gets "ERROR: deadlock detected"

**Frequency in production**: Moderate
**Severity**: P2 — one transaction killed per event, app retries
**Related**: 1.10 Lock Manager

---

## The Symptom

```
ERROR: deadlock detected
DETAIL: Process A waits for ShareLock; blocked by process B.
        Process B waits for ShareLock; blocked by process A.
```

One transaction is aborted. App retries and succeeds. But if it happens constantly, it degrades throughput.

---

## Why This Happens

Two sessions lock the same rows in opposite order:

```
Session A: locks row 1, then tries to lock row 2 (held by B)
Session B: locks row 2, then tries to lock row 1 (held by A)
→ Circular wait → deadlock
```

---

## Diagnosis

```sql
-- Deadlock counter (should be 0 or near 0)
SELECT datname, deadlocks FROM pg_stat_database;

-- Recent deadlocks are in the log with full query details:
-- grep "deadlock" $PGDATA/log/postgresql-*.log | tail -20
```

---

## Fix

Always lock rows in the same order across all code paths:

```sql
-- BAD: order is non-deterministic
UPDATE orders SET status='X' WHERE id IN (2, 1);

-- GOOD: always ascending order
UPDATE orders SET status='X'
WHERE id IN (SELECT id FROM orders WHERE id IN (1,2) ORDER BY id FOR UPDATE);
```

---

## Lab Exercise

```sql
CREATE TABLE dl_test (id int PRIMARY KEY, val text);
INSERT INTO dl_test VALUES (1,'a'),(2,'b');
```

```sql
-- Terminal 1: lock row 1 first
BEGIN;
UPDATE dl_test SET val='T1-row1' WHERE id=1;
-- Wait... don't commit
```

```sql
-- Terminal 2: lock row 2 first, then try row 1
BEGIN;
UPDATE dl_test SET val='T2-row2' WHERE id=2;
-- Now create circular wait:
UPDATE dl_test SET val='T2-row1' WHERE id=1;   -- BLOCKS on T1
```

```sql
-- Terminal 1: try to lock row 2 (T2 holds it)
UPDATE dl_test SET val='T1-row2' WHERE id=2;
-- DEADLOCK — after deadlock_timeout (1s) one transaction is aborted.
-- That session stays connected; only its transaction is rolled back.
```

```sql
-- Check deadlock counter
SELECT datname, deadlocks FROM pg_stat_database WHERE datname=current_database();

-- Clean up
ROLLBACK;   -- in BOTH terminals (the aborted one must roll back before it can do anything else)
DROP TABLE dl_test;
```

**Write down**: Which terminal's transaction was aborted? ___  
What is the fix in application code? ___

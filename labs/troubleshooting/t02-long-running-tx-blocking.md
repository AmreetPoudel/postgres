# T-02 — Long Transaction Blocking Everything (Lock Queue Pile-Up)

**Frequency in production**: Very common
**Severity**: P1 — all queries to a table hang
**Related**: 1.10 Lock Manager, 2.1 postgresql.conf

---

## The Symptom

Suddenly all queries to a table hang. App times out. Monitoring shows connections piling up with wait_event_type = 'Lock'.

---

## Why This Happens

```
Session A (open for 2 hours):
  BEGIN; SELECT * FROM orders;   -- holds AccessShareLock, forgets to commit

Session B:
  ALTER TABLE orders ADD COLUMN x text;
  -- Needs AccessExclusiveLock
  -- BLOCKED by Session A

Session C, D, E... (new app queries):
  SELECT * FROM orders;
  -- ALL queue behind Session B's ALTER
  -- Even though they just want to read
```

Session B's ALTER blocks everything behind it. Session A (idle in transaction) is the root cause.

---

## Diagnosis

```sql
-- 1. Find everything waiting on locks
SELECT pid, wait_event_type, wait_event, state,
       now() - query_start AS duration,
       left(query, 100)    AS query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock'
   OR state = 'idle in transaction'
ORDER BY duration DESC;

-- 2. Find the full blocking chain
-- Who is blocking whom? pg_blocking_pids() understands every lock type,
-- including the table locks behind a DDL pile-up
SELECT
    blocked.pid                  AS blocked_pid,
    left(blocked.query, 60)      AS blocked_query,
    blocking.pid                 AS blocking_pid,
    blocking.state               AS blocking_state,
    now() - blocking.xact_start  AS blocker_age,
    left(blocking.query, 60)     AS blocking_query
FROM pg_stat_activity blocked
CROSS JOIN LATERAL unnest(pg_blocking_pids(blocked.pid)) AS b(pid)
JOIN pg_stat_activity blocking ON blocking.pid = b.pid;

-- The root blockers: they block others but aren't waiting themselves
SELECT pid, state, now() - xact_start AS xact_age, left(query, 60) AS query
FROM pg_stat_activity
WHERE pid IN (SELECT unnest(pg_blocking_pids(pid)) FROM pg_stat_activity)
  AND cardinality(pg_blocking_pids(pid)) = 0;

-- 3. Oldest open transaction (root of all problems)
SELECT pid, usename, state,
       now() - xact_start AS xact_age,
       left(query, 100)   AS query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_age DESC LIMIT 5;
```

---

## Immediate Fix

```sql
-- Kill the root blocker (the idle-in-transaction session; 12345 = its pid
-- from the root-blocker query above)
SELECT pg_terminate_backend(12345);
-- All queued sessions immediately unblock
```

---

## Prevention

```
# postgresql.conf
idle_in_transaction_session_timeout = 5min
lock_timeout = 5s                           # DDL fails fast instead of blocking (applies to
                                            # every statement: many teams set it per role instead)
statement_timeout = 30min
```

For DDL on live tables:
```sql
SET lock_timeout = '3s';
ALTER TABLE orders ADD COLUMN archived boolean DEFAULT false;
-- If lock not acquired in 3s -> error (retry later, not block forever)
```

---

## Lab Exercise

```sql
-- Create a test table
CREATE TABLE lock_lab (id int PRIMARY KEY, val text);
INSERT INTO lock_lab VALUES (1,'a'),(2,'b'),(3,'c');

-- Terminal 1: open idle-in-transaction
BEGIN;
SELECT * FROM lock_lab;
-- Leave open. Don't commit.

-- Terminal 2: try DDL (will block)
-- First set no lock_timeout so it actually blocks:
SET lock_timeout = '0';
ALTER TABLE lock_lab ADD COLUMN extra text;
-- Observe: this hangs

-- Terminal 3: an ordinary read, which now queues behind the ALTER
SELECT * FROM lock_lab;
-- Observe: this hangs too, although Terminal 1's identical read went straight through

-- Terminal 4: diagnose while hanging
SELECT pid, pg_blocking_pids(pid) AS blocked_by, wait_event_type, wait_event,
       state, left(query, 60) AS query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0 OR state = 'idle in transaction';
-- Terminal 3 is blocked by Terminal 2, Terminal 2 by Terminal 1: Terminal 1 is the root

-- Kill the root blocker (Terminal 1's pid from the output above)
SELECT pg_terminate_backend(12345);

-- Terminal 2's ALTER now succeeds, then Terminal 3's SELECT returns
-- Terminal 1 gets: FATAL: terminating connection due to administrator command

-- Cleanup
ALTER TABLE lock_lab DROP COLUMN IF EXISTS extra;
DROP TABLE lock_lab;
```

**Write down**: How many sessions were blocked? ___  
Which PID was the root cause? ___

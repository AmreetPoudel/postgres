# T-01 — Connection Exhaustion: "FATAL: remaining connection slots are reserved"

**Frequency in production**: Very common
**Severity**: P1 — application completely down
**Related**: 2.1 postgresql.conf, 2.4 PgBouncer

---

## The Symptom

Application logs flood with:
```
FATAL: remaining connection slots are reserved for roles with the SUPERUSER attribute
```
(That's the PostgreSQL 16 wording; 15 and older say "reserved for non-replication superuser connections".)
or:
```
FATAL: sorry, too many clients already
```

Users cannot log in. New deployments fail. Alerts fire everywhere.

---

## Why This Happens

```sql
SHOW max_connections;
-- e.g., 100
-- PostgreSQL reserves superuser_reserved_connections (default 3) for admins.
-- When all 97 regular slots are taken, new connections fail.
-- PG16 can reserve more for roles granted pg_use_reserved_connections (reserved_connections).
```

Common causes:
- Application deployed more instances than planned
- Connection leak (connections opened, never closed)
- Long idle transactions holding connections
- PgBouncer is down — app connects directly to PostgreSQL
- Query stuck in long transaction, holding connection

---

## Diagnosis

```sql
-- 1. How many client connections right now? (pg_stat_activity also lists
--    background processes, which don't use max_connections slots)
SELECT count(*) FROM pg_stat_activity WHERE backend_type = 'client backend';

-- 2. What state are they in?
SELECT state, count(*)
FROM pg_stat_activity
WHERE backend_type = 'client backend'
GROUP BY state ORDER BY count(*) DESC;
-- idle                → doing nothing (leak or pool misconfiguration)
-- idle in transaction → in BEGIN but paused (dangerous)
-- active              → running a query

-- 3. Who has been idle longest?
SELECT pid, usename, application_name, client_addr, state,
       now() - state_change AS idle_duration,
       left(query, 80)      AS last_query
FROM pg_stat_activity
WHERE state IN ('idle', 'idle in transaction')
ORDER BY idle_duration DESC LIMIT 20;

-- 4. Where are connections coming from?
SELECT client_addr, count(*)
FROM pg_stat_activity
GROUP BY client_addr ORDER BY count(*) DESC;
```

---

## Immediate Fix

A superuser can still get in through the `superuser_reserved_connections` slots, unless your application itself connects as a superuser and has used those up too (one more reason it shouldn't):

```bash
psql -U postgres -d postgres
```

```sql
-- Kill idle connections older than 10 minutes
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle'
  AND state_change < now() - interval '10 minutes'
  AND pid <> pg_backend_pid();

-- Kill idle-in-transaction older than 5 minutes
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND state_change < now() - interval '5 minutes'
  AND pid <> pg_backend_pid();
```

---

## Root Cause Fix

```
# postgresql.conf
idle_in_transaction_session_timeout = 10min
statement_timeout = 30min
```

Long-term: deploy PgBouncer in transaction pooling mode (see 2.4).

---

## Lab Exercise

```sql
-- Check current max_connections
SHOW max_connections;
SHOW superuser_reserved_connections;

-- Simulate: check how many are actually available
SELECT
    current_setting('max_connections')::int                              AS max_conn,
    current_setting('superuser_reserved_connections')::int               AS reserved,
    current_setting('max_connections')::int
        - current_setting('superuser_reserved_connections')::int         AS regular_available,
    count(*)                                                             AS current_connections,
    current_setting('max_connections')::int
        - current_setting('superuser_reserved_connections')::int
        - count(*)                                                        AS slots_remaining
FROM pg_stat_activity
WHERE backend_type = 'client backend';

-- Set idle timeout and verify it applies
ALTER SYSTEM SET idle_in_transaction_session_timeout = '5min';
SELECT pg_reload_conf();
SHOW idle_in_transaction_session_timeout;

-- Test: open an idle-in-transaction session
-- In another terminal: BEGIN; SELECT 1;  (don't commit)
-- Wait 5 minutes or set timeout to '10s' for the test

-- Clean up
ALTER SYSTEM RESET idle_in_transaction_session_timeout;
SELECT pg_reload_conf();
```

**Write down**: regular_available before your session opened = ___  
What does slots_remaining drop to when you open more connections? ___

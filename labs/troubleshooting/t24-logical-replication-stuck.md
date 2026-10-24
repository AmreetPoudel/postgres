# T-24 — Logical Replication Subscription Stuck

**Frequency**: Moderate
**Severity**: P2 — subscriber falling behind, risk of slot bloat
**Related**: 4.4 Logical Replication, 4.3 Replication Slots

---

## The Symptom

Logical replication lag is growing. `pg_stat_subscription` shows the worker is not advancing. Or the worker pid is NULL (worker crashed).

---

## Why This Happens

1. **Conflict on subscriber**: unique key violation or NOT NULL violation on an INSERT being replicated
2. **Missing replica identity**: a published table with no primary key and no `REPLICA IDENTITY`. The *publisher* then rejects UPDATE and DELETE on it (`cannot update table ... because it does not have a replica identity and publishes updates`): the application gets the error, not the subscriber
3. **Table schema mismatch**: publisher has a column the subscriber doesn't
4. **Subscriber disk full**: can't write WAL
5. **Slot retained too much WAL**: publisher's disk is filling

---

## Diagnosis

```sql
-- On SUBSCRIBER
SELECT subname, pid, received_lsn, last_msg_send_time,
       now() - last_msg_receipt_time AS since_last_msg
FROM pg_stat_subscription;
-- pid = NULL means worker is not running

-- Check subscription errors
SELECT * FROM pg_stat_subscription_stats;   -- PG15+

-- On PUBLISHER: check slot lag
SELECT slot_name, active,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS lag
FROM pg_replication_slots WHERE slot_type = 'logical';
```

```bash
# Check subscriber logs for the error
grep "ERROR\|FATAL" "$PGDATA"/log/postgresql-*.log | grep -i "subscription\|replica" | tail -20
```

---

## Fix: Worker Crashed

```sql
-- Disable and re-enable the subscription
ALTER SUBSCRIPTION my_sub DISABLE;
ALTER SUBSCRIPTION my_sub ENABLE;

-- Check if worker came back
SELECT pid, received_lsn FROM pg_stat_subscription;
```

---

## Fix: Conflict

```sql
-- On SUBSCRIBER: find the conflicting row and resolve it
-- Example: unique violation on id=42
DELETE FROM local_table WHERE id = 42;   -- or UPDATE to resolve conflict

-- Then the subscription will resume automatically

-- PG15+: or skip the failing remote transaction entirely (its changes are lost).
-- Take the LSN from "... finished at <LSN>" in the subscriber's error log:
ALTER SUBSCRIPTION my_sub SKIP (lsn = '0/14C0378');

-- PG15+: stop retrying in a loop after an error, and wait for you instead
ALTER SUBSCRIPTION my_sub SET (disable_on_error = true);
```

---

## Fix: Missing Replica Identity

```sql
-- On PUBLISHER: published tables with no usable replica identity. 'n' = nothing;
-- 'd' (default) only works if the table has a primary key
SELECT pt.schemaname, pt.tablename, c.relreplident
FROM pg_publication_tables pt
JOIN pg_class c ON c.oid = format('%I.%I', pt.schemaname, pt.tablename)::regclass
WHERE pt.pubname = 'my_pub'
  AND (c.relreplident = 'n'
       OR (c.relreplident = 'd'
           AND NOT EXISTS (SELECT 1 FROM pg_index i WHERE i.indrelid = c.oid AND i.indisprimary)));

-- Fix: add replica identity
ALTER TABLE problem_table REPLICA IDENTITY FULL;
-- or add a primary key:
ALTER TABLE problem_table ADD PRIMARY KEY (id);
```

---

## Lab Exercise

```sql
-- Prerequisite: wal_level = logical, which needs a restart:
--   ALTER SYSTEM SET wal_level = logical;   -- then restart PostgreSQL
SHOW wal_level;   -- must say logical

-- Setup: simple logical replication (publisher and subscriber in same cluster, different DBs)
CREATE DATABASE pub_db;
CREATE DATABASE sub_db;
```

```sql
-- On pub_db
\c pub_db
CREATE TABLE pub_test (id serial PRIMARY KEY, data text);
INSERT INTO pub_test (data) VALUES ('row1'),('row2'),('row3');
CREATE PUBLICATION my_pub FOR TABLE pub_test;
```

```sql
-- Still on pub_db: create the slot first. Within ONE cluster, CREATE SUBSCRIPTION
-- hangs if it has to create the slot itself (a documented restriction).
SELECT pg_create_logical_replication_slot('my_sub', 'pgoutput');

-- On sub_db
\c sub_db
CREATE TABLE pub_test (id serial PRIMARY KEY, data text);
CREATE SUBSCRIPTION my_sub
    CONNECTION 'host=127.0.0.1 port=5432 dbname=pub_db user=postgres'   -- add password=... if pg_hba.conf asks for one
    PUBLICATION my_pub
    WITH (create_slot = false);

-- Check sync started
SELECT subname, pid FROM pg_stat_subscription;
SELECT * FROM pub_test;   -- should see 3 rows
```

```sql
-- Simulate conflict: insert same id on subscriber
\c sub_db
INSERT INTO pub_test (id, data) VALUES (100, 'conflict row');

-- On pub_db: insert id=100 to trigger conflict
\c pub_db
INSERT INTO pub_test (id, data) VALUES (100, 'from publisher');
-- This should cause conflict on subscriber
```

```sql
-- On sub_db: check subscription worker
SELECT subname, pid FROM pg_stat_subscription;
-- pid may be NULL (worker crashed)

-- Watch the error count climb while the worker retries
SELECT subname, apply_error_count FROM pg_stat_subscription_stats;

-- Resolve conflict: remove the local row that's in the way
\c sub_db
DELETE FROM pub_test WHERE id = 100;
-- No restart needed: the worker's next retry (within seconds) succeeds
SELECT pg_sleep(10);
SELECT * FROM pub_test WHERE id = 100;          -- now the publisher's row
SELECT subname, pid FROM pg_stat_subscription;  -- worker running
```

```sql
-- Cleanup
\c sub_db
DROP SUBSCRIPTION my_sub;
\c pub_db
DROP PUBLICATION my_pub;
DROP TABLE pub_test;
\c sub_db
DROP TABLE pub_test;
\c postgres
DROP DATABASE pub_db;
DROP DATABASE sub_db;
```

**Write down**: What caused the subscription worker to stop? ___  
How did you resume it? ___

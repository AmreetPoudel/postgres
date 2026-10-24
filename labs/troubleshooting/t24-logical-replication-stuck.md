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
2. **Worker crashed**: replica identity missing (no primary key), can't apply UPDATE/DELETE
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
grep "ERROR\|FATAL" "$PGDATA/log/postgresql-*.log" | grep -i "subscription\|replica" | tail -20
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
```

---

## Fix: Missing Replica Identity

```sql
-- On PUBLISHER: find tables without primary key
SELECT relname FROM pg_class
WHERE relkind = 'r'
  AND relreplident = 'n'    -- 'n' = nothing (can't replicate UPDATE/DELETE)
  AND relname IN (SELECT tablename FROM pg_publication_tables WHERE pubname = 'my_pub');

-- Fix: add replica identity
ALTER TABLE problem_table REPLICA IDENTITY FULL;
-- or add a primary key:
ALTER TABLE problem_table ADD PRIMARY KEY (id);
```

---

## Lab Exercise

```sql
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
-- On sub_db
\c sub_db
CREATE TABLE pub_test (id serial PRIMARY KEY, data text);
CREATE SUBSCRIPTION my_sub
    CONNECTION 'host=127.0.0.1 port=5432 dbname=pub_db user=postgres'
    PUBLICATION my_pub;

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

-- Resolve conflict
\c sub_db
DELETE FROM pub_test WHERE id = 100;
ALTER SUBSCRIPTION my_sub DISABLE;
ALTER SUBSCRIPTION my_sub ENABLE;
SELECT subname, pid FROM pg_stat_subscription;  -- worker should be back
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

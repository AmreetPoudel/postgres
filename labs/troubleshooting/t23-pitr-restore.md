# T-23 — Performing PITR After Accidental Data Deletion

**Frequency**: Occasional (data accidents happen)
**Severity**: P1 — data loss, need to recover from backup
**Related**: 3.3 PITR, 3.4 pgBackRest

---

## The Scenario

At 14:32:05, a developer ran:
```sql
DELETE FROM orders WHERE status = 'completed';
-- 2.3 million rows deleted. Should have had a WHERE clause.
```

You need to restore the data as of 14:31:59, 6 seconds before the accident.

---

## Diagnosis Before Restore

```sql
-- Find the exact time of the accidental operation from the logs, if statements
-- or slow queries are logged:
--   grep "DELETE FROM orders" $PGDATA/log/postgresql-*.log

-- More precise: find the transaction itself in the WAL, then stop just before it.
-- pg_waldump (a command-line tool) prints each record with its XID and LSN; look
-- for the DELETE's commit record around 14:32:05:
--   pg_waldump -p $PGDATA/pg_wal --start=<an LSN from before the accident> | grep COMMIT
-- then recover with recovery_target_xid = '<that xid>' and
-- recovery_target_inclusive = off (pgBackRest: --type=xid --target-exclusive)
```

---

## Option A: In-Place PITR with pgBackRest

**Warning:** an in-place PITR rewinds the *whole* cluster. Everything committed after 14:31:59, every legitimate order since the accident, is lost too. Prefer Option B unless the cluster is useless without an immediate rewind.

```bash
# 1. Stop the running cluster
pg_ctl stop -D /var/lib/postgresql/16/main -m fast

# 2. Move current data directory aside, and create an empty one
mv /var/lib/postgresql/16/main /var/lib/postgresql/16/main.broken
install -d -o postgres -g postgres -m 700 /var/lib/postgresql/16/main

# 3. Restore from backup to a point BEFORE the accident
# Replace timestamp with 1 minute before the accident:
pgbackrest --stanza=mydb \
           --type=time \
           --target="2026-10-14 14:31:59" \
           --target-action=promote \
           restore

# 4. Start PostgreSQL — it will apply WAL until 14:31:59, then promote
pg_ctl start -D /var/lib/postgresql/16/main

# 5. Verify the data is back
psql -c "SELECT count(*) FROM orders WHERE status='completed';"
# Should show the 2.3M rows
```

---

## Option B: Restore to Separate Server (Safer)

```bash
# 1. PITR-restore the backup onto a separate server, to just before the accident
#    (the same restore command as Option A, run there, plus --archive-mode=off)

# 2. On the recovery server, keep only the rows that were deleted:
psql -h recovery-server -d mydb -c "CREATE TABLE orders_recovered AS
    SELECT * FROM orders WHERE status = 'completed';"

# 3. Copy that table to production, as a separate table:
pg_dump -h recovery-server -d mydb -t orders_recovered | psql -h production -d mydb

# 4. Put the rows back without touching anything written since the accident:
psql -h production -d mydb -c "INSERT INTO orders SELECT * FROM orders_recovered
    ON CONFLICT (id) DO NOTHING;"
```

Restoring the whole `orders` table instead would collide with the rows that were never deleted, and would throw away everything written since the accident.

---

## Lab Exercise

```sql
-- Create a "production" table with important data
CREATE TABLE pitr_lab (
    id         serial PRIMARY KEY,
    important  text DEFAULT md5(random()::text),
    created_at timestamp DEFAULT now()
);
INSERT INTO pitr_lab (important) SELECT md5(i::text) FROM generate_series(1, 10000) i;

-- Note current state
SELECT count(*) FROM pitr_lab;    -- 10,000
SELECT max(created_at) FROM pitr_lab;

-- Note the CURRENT time (this is our recovery target)
SELECT now() AS safe_point;       -- write this down: ___
```

```sql
-- Wait 5 seconds, then "accidentally" delete everything
SELECT pg_sleep(5);
DELETE FROM pitr_lab;

-- Verify data is gone
SELECT count(*) FROM pitr_lab;    -- 0
```

```sql
-- If you have pgBackRest configured with WAL archiving, restore to "safe_point"
-- If not: practice by querying pg_wal_lsn_diff to understand the gap

SELECT pg_current_wal_lsn() AS current_lsn;

-- What you WOULD do with pgBackRest:
-- pgbackrest --stanza=mydb --type=time --target='SAFE_POINT_TIMESTAMP' restore
```

```sql
-- Cleanup
DROP TABLE pitr_lab;
```

**Write down**: What was the safe_point timestamp? ___  
What command restores to that exact point with pgBackRest? ___

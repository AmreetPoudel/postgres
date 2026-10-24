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
-- Find the exact time of the accidental operation from logs
-- grep "DELETE FROM orders" $PGDATA/log/postgresql-*.log

-- Or find the WAL LSN at accident time
SELECT pg_current_wal_lsn();   -- note current LSN
-- In logs: look for "duration:" entry corresponding to the DELETE
```

---

## Option A: PITR with pgBackRest (Production Standard)

```bash
# 1. Stop the running cluster
pg_ctl stop -D /var/lib/postgresql/16/main -m fast

# 2. Move current data directory aside
mv /var/lib/postgresql/16/main /var/lib/postgresql/16/main.broken

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
# Restore to separate server (leaves production untouched)
# Then extract just the deleted data:
pg_dump -t orders -f /tmp/recovered_orders.sql recovery-server

# Import into production:
psql -d production_db -f /tmp/recovered_orders.sql
```

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

# T-11 — Transaction ID Wraparound Emergency

**Frequency**: Rare but catastrophic
**Severity**: P0 — the database stops accepting new write transactions
**Related**: 1.7 VACUUM, 2.5 Autovacuum Tuning

---

## The Symptom

```
WARNING: database "mydb" must be vacuumed within 39,000,000 transactions
```
Then, if ignored:
```
ERROR: database is not accepting commands to avoid wraparound data loss
```
The database refuses every command that needs a new transaction ID, so all writes fail. Read-only queries still work, and so does VACUUM. It's protecting itself.

The HINT that comes with the error tells you to stop the postmaster and vacuum in single-user mode. The PostgreSQL 16 documentation says that's no longer necessary or desirable: ignore it and follow the procedure below.

---

## Why This Happens

Transaction IDs are 32-bit integers (max ~2.1 billion). PostgreSQL uses modular arithmetic — every row is marked with the XID that created it. If the counter wraps around, old rows appear to be from "the future" and become invisible — silent data corruption.

PostgreSQL prevents this by freezing old tuples during VACUUM. Autovacuum normally handles this, but fails if:
- Something holds the oldest XID back: a long-running or idle-in-transaction session, an old prepared transaction, a stale replication slot, a standby with `hot_standby_feedback`
- A table is too large, or autovacuum too throttled, to finish freezing in time
- It's turned off: anti-wraparound vacuums still start, but only at `autovacuum_freeze_max_age`, with no routine freezing before that

---

## Diagnosis

```sql
-- XID age per database
SELECT datname,
       age(datfrozenxid)   AS xid_age,
       round(age(datfrozenxid) / 2000000000.0 * 100, 2) AS pct_to_danger
FROM pg_database ORDER BY xid_age DESC;

-- Per-table (find the worst offender; TOAST tables and materialized views count too)
SELECT relname, age(relfrozenxid) AS xid_age
FROM pg_class WHERE relkind IN ('r', 'm', 't') AND relfrozenxid != 0
ORDER BY xid_age DESC LIMIT 10;

-- PostgreSQL warns at 40M XIDs from the limit and stops assigning XIDs at 3M
-- Critical: age > 1,000,000,000 (VACUUM's failsafe engages at 1,600,000,000)

-- What holds the oldest XID back? Fix these FIRST, or VACUUM can't advance anything
SELECT 'session snapshot' AS holder, pid::text AS id, age(backend_xmin) AS xmin_age, state, left(query, 40) AS detail
FROM pg_stat_activity WHERE backend_xmin IS NOT NULL
UNION ALL
SELECT 'session xid', pid::text, age(backend_xid), state, left(query, 40)
FROM pg_stat_activity WHERE backend_xid IS NOT NULL
UNION ALL
SELECT 'replication slot', slot_name::text, greatest(age(xmin), age(catalog_xmin)), active::text, slot_type
FROM pg_replication_slots WHERE xmin IS NOT NULL OR catalog_xmin IS NOT NULL
UNION ALL
SELECT 'prepared xact', gid, age(transaction), owner::text, database::text
FROM pg_prepared_xacts
ORDER BY xmin_age DESC;
```

---

## Emergency Fix

The PostgreSQL 16 procedure, in this order:

1. **Resolve old prepared transactions** (`pg_prepared_xacts`): `COMMIT PREPARED` or `ROLLBACK PREPARED` them.
2. **End long-running transactions** (large `age(backend_xid)` or `age(backend_xmin)` in `pg_stat_activity`): commit, roll back, or `pg_terminate_backend()`.
3. **Drop stale replication slots** (large `age(xmin)` or `age(catalog_xmin)` in `pg_replication_slots`).
4. **VACUUM the affected database**, as a superuser so the system catalogs get processed too. Plain VACUUM: not `VACUUM FREEZE` (more work than needed) and not `VACUUM FULL` (it needs a new XID, which is exactly what you can't get).
5. Once writes work again, fix autovacuum so it never gets this far (2.5).

```bash
# Worst tables first (from the per-table query above)...
psql -d mydb -c "VACUUM (VERBOSE) worst_table_name;"

# ...or the whole database, 4 tables at a time
vacuumdb -d mydb --jobs=4 --verbose
```

```sql
-- While VACUUM runs, watch XID age drop:
SELECT relname, age(relfrozenxid) AS xid_age
FROM pg_class WHERE relkind IN ('r', 'm', 't') AND relfrozenxid != 0
ORDER BY xid_age DESC LIMIT 5;
```

---

## Prevention

```
# postgresql.conf
autovacuum_freeze_max_age   = 100000000   # force freezing at 100M XIDs of age (needs a restart)
vacuum_freeze_table_age     = 80000000    # ordinary vacuums scan aggressively from 80M
log_autovacuum_min_duration = 500         # log autovacuum runs that take over 500ms
```

```sql
-- Weekly alert query
SELECT datname, age(datfrozenxid) AS xid_age
FROM pg_database WHERE age(datfrozenxid) > 500000000;
-- If any row returns → investigate immediately
```

---

## Lab Exercise

```sql
-- Check your current XID age
SELECT datname, age(datfrozenxid) AS xid_age
FROM pg_database WHERE datname = current_database();

-- Simulate what autovacuum_freeze_max_age does:
SHOW autovacuum_freeze_max_age;
-- Default: 200000000 (200M)

-- See current oldest table
SELECT relname, age(relfrozenxid) AS xid_age
FROM pg_class WHERE relkind='r' AND relfrozenxid != 0
ORDER BY xid_age DESC LIMIT 5;

-- Run VACUUM FREEZE on the oldest one:
VACUUM FREEZE VERBOSE pg_class;

-- Check age dropped:
SELECT relname, age(relfrozenxid) AS xid_age
FROM pg_class WHERE relname = 'pg_class';
```

**Write down**: Before FREEZE, age of pg_class: ___  
After FREEZE: ___

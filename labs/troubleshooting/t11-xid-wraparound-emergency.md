# T-11 — Transaction ID Wraparound Emergency

**Frequency**: Rare but catastrophic
**Severity**: P0 — database goes read-only then shuts down
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
The database refuses all writes. It's protecting itself.

---

## Why This Happens

Transaction IDs are 32-bit integers (max ~2.1 billion). PostgreSQL uses modular arithmetic — every row is marked with the XID that created it. If the counter wraps around, old rows appear to be from "the future" and become invisible — silent data corruption.

PostgreSQL prevents this by freezing old tuples during VACUUM. Autovacuum normally handles this, but fails if:
- It's turned off
- Long-running transactions block it
- Table is too large and autovacuum can't keep pace

---

## Diagnosis

```sql
-- XID age per database
SELECT datname,
       age(datfrozenxid)   AS xid_age,
       round(age(datfrozenxid) / 2000000000.0 * 100, 2) AS pct_to_danger
FROM pg_database ORDER BY xid_age DESC;

-- Per-table (find the worst offender)
SELECT relname, age(relfrozenxid) AS xid_age
FROM pg_class WHERE relkind = 'r' AND relfrozenxid != 0
ORDER BY xid_age DESC LIMIT 10;

-- Threshold: PostgreSQL warns at 40M from limit, shuts down at 3M from limit
-- Danger zone: age > 1,500,000,000
```

---

## Emergency Fix

```bash
# Stop as much write traffic as possible — you need VACUUM to run
# autovacuum for wraparound runs even when pg is in read-only warning mode

# Manual aggressive VACUUM FREEZE on the worst table
psql -c "VACUUM FREEZE VERBOSE worst_table_name;"

# If the whole DB is close:
vacuumdb --all --freeze --jobs=4 --verbose
```

```sql
-- While VACUUM FREEZE runs, watch XID age drop:
SELECT relname, age(relfrozenxid) AS xid_age
FROM pg_class WHERE relkind='r' AND relfrozenxid != 0
ORDER BY xid_age DESC LIMIT 5;
```

---

## Prevention

```
# postgresql.conf
autovacuum_freeze_max_age   = 100000000   -- freeze before 100M XIDs age
vacuum_freeze_table_age     = 80000000
log_autovacuum_min_duration = 500         -- see when wraparound vacuums run
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

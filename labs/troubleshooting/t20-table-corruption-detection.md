# T-20 — Detecting Table Corruption

**Frequency**: Rare but career-defining when it happens
**Severity**: P0 — data loss potential
**Related**: 1.4 Storage Layout

---

## The Symptom

```
ERROR: invalid page in block 42 of relation base/16384/12345
```
or queries return inconsistent results. Or `pg_dump` hangs on a specific table.

---

## Why This Happens

- Disk/SSD failure writing a page
- Storage controller bug / firmware bug
- Power loss during write without proper fsync
- Bugs in PostgreSQL (rare) or in storage hardware
- VM disk images with checksum disabled

---

## Diagnosis

```sql
-- Enable data checksums (must be done at initdb time or pg_checksums offline)
-- Check if enabled:
SHOW data_checksums;
-- 'on' = checksums will catch corruption
-- 'off' = corruption may go undetected

-- Enable pg_checksums on an offline cluster (PG12+):
-- pg_checksums --enable -D "$PGDATA"  (offline only, takes a while)
```

```bash
# Online corruption check with amcheck (checks index structure)
psql -c "CREATE EXTENSION IF NOT EXISTS amcheck;"
psql -c "SELECT bt_index_check('orders_pkey');"    -- check primary key index
psql -c "SELECT bt_index_parent_check('orders_pkey', true);"  -- full check with heap

# Find pg_relation_filepath for the table
psql -c "SELECT pg_relation_filepath('orders');"
# Returns something like: base/16384/12345

# Direct block check (look for 'invalid page' errors)
pg_filedump -i "$PGDATA/base/16384/12345" 2>&1 | head -20
```

---

## Fix

There is no automatic fix for corruption. Options:
1. Restore from backup (cleanest)
2. `pg_dump --ignore-version` and reimport clean data
3. For index corruption: `REINDEX TABLE table_name` (rebuild from heap)

```sql
-- If only index is corrupt (heap is fine):
REINDEX TABLE orders;

-- Then verify:
SELECT bt_index_check('orders_pkey');
```

---

## Prevention

```bash
# ALWAYS initialize PostgreSQL with checksums
initdb -D /var/lib/postgresql/16/main --data-checksums

# Enable checksums on existing cluster (offline)
pg_ctl stop -D "$PGDATA" -m fast
pg_checksums --enable -D "$PGDATA"
pg_ctl start -D "$PGDATA"
```

---

## Lab Exercise

```sql
-- Check if checksums are enabled
SHOW data_checksums;

-- Install amcheck
CREATE EXTENSION IF NOT EXISTS amcheck;

-- Check all user indexes for corruption
SELECT relname, pg_relation_size(oid) AS size
FROM pg_class WHERE relkind = 'i' AND relname NOT LIKE 'pg_%';

-- Check your biggest index
SELECT bt_index_check(
    (SELECT oid FROM pg_class WHERE relname = 'large_table_pkey')
);
-- Returns nothing on success, raises error if corrupt

-- Check with heap correlation
SELECT bt_index_parent_check(
    'orders_pkey',   -- replace with an existing index name
    true
);
```

```bash
# Find relation file path
psql -c "SELECT relname, pg_relation_filepath(oid)
         FROM pg_class WHERE relkind='r' AND relname = 'pg_class';"

# Verify file exists and is not zero-length
ls -lh "$PGDATA/$(psql -t -c "SELECT pg_relation_filepath('pg_class');")"
```

**Write down**: Are checksums enabled on your PostgreSQL? ___  
What command enables checksums on an existing cluster? ___

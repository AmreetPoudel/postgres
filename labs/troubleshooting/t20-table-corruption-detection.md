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
# Light check: takes only an ACCESS SHARE lock, safe on a busy server
psql -c "SELECT bt_index_check('orders_pkey', true);"
# Thorough check (parent/child links too): takes a SHARE lock that BLOCKS WRITES
psql -c "SELECT bt_index_parent_check('orders_pkey', true);"

# PG14+: pg_amcheck checks tables and their B-tree indexes from the command line
pg_amcheck -d mydb --heapallindexed --progress

# Find pg_relation_filepath for the table
psql -c "SELECT pg_relation_filepath('orders');"
# Returns something like: base/16384/12345

# Direct block check (look for 'invalid page' errors). pg_filedump isn't part of
# core PostgreSQL: install it from your distribution or the PGDG repository
pg_filedump -i "$PGDATA/base/16384/12345" 2>&1 | head -20
```

---

## Fix

There is no automatic fix for corruption. Options:
1. Restore from backup (cleanest), or fail over to a standby on separate storage after checking it: a physical standby often doesn't share hardware-caused damage
2. Salvage what's readable with pg_dump, table by table, into a fresh cluster. As a last resort, `zero_damaged_pages = on` lets reads skip broken pages by wiping their contents: only ever on a copy
3. For index corruption: `REINDEX TABLE CONCURRENTLY table_name` (rebuilds from the heap)

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

-- Check every valid B-tree index in this database (the amcheck docs' own pattern)
SELECT c.relname, bt_index_check(c.oid)
FROM pg_index i
JOIN pg_class c      ON c.oid = i.indexrelid
JOIN pg_am am        ON am.oid = c.relam
JOIN pg_namespace n  ON n.oid = c.relnamespace
WHERE am.amname = 'btree'
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
  AND c.relpersistence <> 't'
  AND i.indisready AND i.indisvalid;
-- An empty result per index means OK; corruption raises an error

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
ls -lh "$PGDATA/$(psql -At -c "SELECT pg_relation_filepath('pg_class');")"
```

**Write down**: Are checksums enabled on your PostgreSQL? ___  
What command enables checksums on an existing cluster? ___

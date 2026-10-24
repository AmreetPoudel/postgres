# T-12 — Index Bloat: Indexes Growing Without Bound

**Frequency**: Common on tables with heavy UPDATE/DELETE
**Severity**: P2 — queries slow, disk fills, backup times grow
**Related**: 1.9 Index Internals, 1.7 VACUUM

---

## The Symptom

Table is 10GB. Its indexes are 40GB. Queries using indexes are slow despite good plans. Index scan reads far more blocks than expected.

---

## Why This Happens

VACUUM removes dead index entries, but the space they free can only be reused by new keys that sort into the *same* page, and a page is only handed back for reuse once it's completely empty. On a table with heavy random updates, index pages end up half empty: the index grows, and every index scan reads more pages than it needs. (B-tree deduplication in PG13 and bottom-up deletion in PG14 reduce this a lot, but not to zero.)

---

## Diagnosis

```sql
-- Biggest indexes next to their tables (a large index-to-table ratio is a hint)
SELECT schemaname, relname AS table_name, indexrelname AS index_name, idx_scan,
       pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
       pg_size_pretty(pg_relation_size(relid))      AS table_size
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC LIMIT 10;

-- Measure it properly with pgstatindex() from the pgstattuple extension (B-tree)
CREATE EXTENSION IF NOT EXISTS pgstattuple;
SELECT avg_leaf_density, leaf_fragmentation FROM pgstatindex('orders_pkey');
-- A freshly built index (CREATE INDEX or REINDEX) sits near 90% leaf density, its
-- fillfactor; one grown by random inserts typically settles around 60-70%.
-- Well below 50% means a REINDEX would shrink it a lot.
```

---

## Fix

```sql
-- Online reindex (PG12+, does not lock reads/writes)
REINDEX INDEX CONCURRENTLY orders_status_idx;

-- Or reindex all on a table
REINDEX TABLE CONCURRENTLY orders;
```

---

## Lab Exercise

```sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;

CREATE TABLE idx_bloat_lab (
    id   serial PRIMARY KEY,
    val  int,
    data text DEFAULT md5(random()::text)
);
CREATE INDEX ON idx_bloat_lab(val);

INSERT INTO idx_bloat_lab(val)
SELECT (random()*1000000)::int FROM generate_series(1, 300000);

-- Baseline
SELECT pg_size_pretty(pg_relation_size('idx_bloat_lab_val_idx')) AS before_size;
SELECT avg_leaf_density, leaf_fragmentation FROM pgstatindex('idx_bloat_lab_val_idx');
```

```sql
-- Create index bloat: update all val entries repeatedly
UPDATE idx_bloat_lab SET val = (random()*1000000)::int;
UPDATE idx_bloat_lab SET val = (random()*1000000)::int;
VACUUM idx_bloat_lab;  -- removes the dead entries, but the index stays big and half empty

-- Measure bloat
SELECT pg_size_pretty(pg_relation_size('idx_bloat_lab_val_idx')) AS bloated_size;
SELECT avg_leaf_density, leaf_fragmentation FROM pgstatindex('idx_bloat_lab_val_idx');
-- (pgstattuple's dead_tuple_percent reads ~0 here: VACUUM removed the dead entries.
--  The bloat is free space, which is what leaf density shows.)
```

```sql
-- Fix with REINDEX CONCURRENTLY
REINDEX INDEX CONCURRENTLY idx_bloat_lab_val_idx;

-- Measure after
SELECT pg_size_pretty(pg_relation_size('idx_bloat_lab_val_idx')) AS after_size;
SELECT avg_leaf_density, leaf_fragmentation FROM pgstatindex('idx_bloat_lab_val_idx');

DROP TABLE idx_bloat_lab;
```

**Write down**: Index size before bloat: ___ after bloat: ___ after REINDEX: ___  
avg_leaf_density before and after REINDEX: ___

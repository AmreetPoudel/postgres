# T-12 — Index Bloat: Indexes Growing Without Bound

**Frequency**: Common on tables with heavy UPDATE/DELETE
**Severity**: P2 — queries slow, disk fills, backup times grow
**Related**: 1.9 Index Internals, 1.7 VACUUM

---

## The Symptom

Table is 10GB. Its indexes are 40GB. Queries using indexes are slow despite good plans. Index scan reads far more blocks than expected.

---

## Why This Happens

VACUUM cleans dead tuples in heap pages and marks dead index entries as reusable. But deleted index entries are not immediately reclaimed — only removed when a page becomes entirely dead. On a table with heavy random updates, index pages become sparse (many dead pointers, few live ones).

---

## Diagnosis

```sql
-- Index bloat estimation
SELECT
    t.tablename,
    i.indexname,
    pg_size_pretty(pg_relation_size(i.indexrelid)) AS index_size,
    pg_size_pretty(pg_relation_size(t.tableid))    AS table_size,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_tables t
JOIN pg_indexes i    ON i.tablename = t.tablename
JOIN pg_stat_user_indexes ui ON ui.indexrelname = i.indexname
JOIN pg_index pi ON pi.indexrelid = ui.indexrelid
WHERE t.schemaname = 'public'
ORDER BY pg_relation_size(i.indexrelid) DESC LIMIT 10;

-- pgstattuple extension (more accurate bloat estimate)
CREATE EXTENSION IF NOT EXISTS pgstattuple;
SELECT * FROM pgstattuple('orders_pkey');
-- Look at: dead_tuple_percent, free_percent
-- dead_tuple_percent > 20% → significant bloat
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
SELECT dead_tuple_percent, free_percent
FROM pgstattuple('idx_bloat_lab_val_idx');
```

```sql
-- Create index bloat: update all val entries repeatedly
UPDATE idx_bloat_lab SET val = (random()*1000000)::int;
UPDATE idx_bloat_lab SET val = (random()*1000000)::int;
VACUUM idx_bloat_lab;  -- cleans heap but not all index bloat

-- Measure bloat
SELECT pg_size_pretty(pg_relation_size('idx_bloat_lab_val_idx')) AS bloated_size;
SELECT dead_tuple_percent, free_percent FROM pgstattuple('idx_bloat_lab_val_idx');
```

```sql
-- Fix with REINDEX CONCURRENTLY
REINDEX INDEX CONCURRENTLY idx_bloat_lab_val_idx;

-- Measure after
SELECT pg_size_pretty(pg_relation_size('idx_bloat_lab_val_idx')) AS after_size;
SELECT dead_tuple_percent, free_percent FROM pgstattuple('idx_bloat_lab_val_idx');

DROP TABLE idx_bloat_lab;
```

**Write down**: Index size before bloat: ___ after bloat: ___ after REINDEX: ___  
dead_tuple_percent before fix: ___

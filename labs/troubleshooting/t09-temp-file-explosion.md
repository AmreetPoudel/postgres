# T-09 — Temp File Explosion: Disk Full from Query Spills

**Frequency in production**: Moderate
**Severity**: P1 if disk fills — all writes stop
**Related**: 1.3 Memory Architecture (work_mem), 2.1 postgresql.conf

---

## The Symptom

Disk fills up. `$PGDATA/base/pgsql_tmp/` is full of huge files. Log shows:
```
LOG: temporary file: path "base/pgsql_tmp/pgsql_tmp12345.0", size 10737418240
```

---

## Why This Happens

When sort/hash doesn't fit in `work_mem`, PostgreSQL spills to a temp file in `$PGDATA/base/pgsql_tmp/`. With many concurrent analytics queries and low work_mem, this fills the disk fast.

---

## Diagnosis

```sql
-- Enable temp file logging
ALTER SYSTEM SET log_temp_files = 0;   -- log all temp files
SELECT pg_reload_conf();

-- Check temp file stats
SELECT datname, temp_files, pg_size_pretty(temp_bytes) AS temp_size
FROM pg_stat_database ORDER BY temp_bytes DESC;
```

```bash
du -sh "$PGDATA/base/pgsql_tmp/"
ls -lh "$PGDATA/base/pgsql_tmp/"
```

---

## Fix

```sql
-- Per session for analytics workloads
SET work_mem = '1GB';
-- Run heavy query
RESET work_mem;
```

```sql
-- Move temp files to a separate disk
CREATE TABLESPACE scratch LOCATION '/mnt/scratch/pgsql_tmp';

-- postgresql.conf
-- temp_tablespaces = 'scratch'
```

---

## Lab Exercise

```sql
ALTER SYSTEM SET log_temp_files = 0;
SELECT pg_reload_conf();

-- Create large dataset
CREATE TABLE sort_lab AS
SELECT (random()*10000000)::int AS k, md5(random()::text) AS v
FROM generate_series(1, 3000000);

-- Force temp file with tiny work_mem
SET work_mem = '1MB';

EXPLAIN (ANALYZE, BUFFERS)
SELECT k, v FROM sort_lab ORDER BY k LIMIT 100;
-- Look for: Sort Method: external merge Disk: xxxMB

-- Check temp stats
SELECT datname, temp_files, pg_size_pretty(temp_bytes)
FROM pg_stat_database WHERE datname = current_database();
```

```sql
-- Fix with adequate work_mem
SET work_mem = '512MB';
EXPLAIN (ANALYZE, BUFFERS)
SELECT k, v FROM sort_lab ORDER BY k LIMIT 100;
-- Sort Method: quicksort Memory: xxxkB

RESET work_mem;
DROP TABLE sort_lab;
ALTER SYSTEM RESET log_temp_files;
SELECT pg_reload_conf();
```

**Write down**: Sort method at 1MB work_mem: ___  
Sort method at 512MB work_mem: ___  
Temp files generated: ___

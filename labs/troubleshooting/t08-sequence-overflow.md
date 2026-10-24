# T-08 — Sequence Overflow: "integer out of range"

**Frequency in production**: Moderate — devastating when it hits
**Severity**: P0 — all INSERTs fail to this table
**Related**: Data type fundamentals

---

## The Symptom

```
ERROR:  nextval: reached maximum value of sequence "orders_id_seq" (2147483647)
```
or, when the sequence is `bigint` but the column is still `integer`:
```
ERROR:  integer out of range
```

All INSERTs to a specific table fail. Completely.

---

## Why This Happens

`serial` (int) max = 2,147,483,647. Once the sequence hits that, every new INSERT fails.
High-volume tables, tables with many rollbacks (sequence increments even on rollback), or old tables using `serial` instead of `bigserial`. Tables created before PostgreSQL 10 often have a bigint sequence feeding an integer column, so the column overflows first.

---

## Diagnosis

```sql
-- Sequences, and how close each is to its own limit
SELECT schemaname, sequencename, data_type, last_value, max_value,
       round(100.0 * last_value / max_value, 2) AS pct_used
FROM pg_sequences
WHERE last_value IS NOT NULL            -- NULL = never used (or no permission to see it)
ORDER BY pct_used DESC;

-- What really matters: integer columns fed by a sequence (serial or identity),
-- measured against the COLUMN's limit, which a bigint sequence doesn't share
SELECT c.relname AS table_name, a.attname AS column_name,
       format_type(a.atttypid, a.atttypmod) AS column_type, s.last_value,
       round(100.0 * s.last_value /
             CASE a.atttypid WHEN 'int4'::regtype THEN 2147483647 ELSE 32767 END, 2) AS pct_of_column_max
FROM pg_depend d
JOIN pg_class seq    ON seq.oid = d.objid AND seq.relkind = 'S'
JOIN pg_class c      ON c.oid = d.refobjid
JOIN pg_attribute a  ON a.attrelid = d.refobjid AND a.attnum = d.refobjsubid
JOIN pg_sequences s  ON s.schemaname = seq.relnamespace::regnamespace::text
                    AND s.sequencename = seq.relname
WHERE d.classid = 'pg_class'::regclass AND d.refclassid = 'pg_class'::regclass
  AND d.deptype IN ('a', 'i')                     -- serial ('a') or identity ('i')
  AND a.atttypid IN ('int2'::regtype, 'int4'::regtype)
ORDER BY pct_of_column_max DESC NULLS LAST;

-- Alert: pct_used > 75%
-- Emergency: pct_used > 90%
```

---

## Immediate Fix

```sql
-- The real fix: the column must become bigint, and so must every column that
-- references it (e.g. order_items.order_id). This rewrites the table and its indexes
-- under an ACCESS EXCLUSIVE lock: minutes to hours of downtime on a big table.
ALTER TABLE orders ALTER COLUMN id TYPE bigint;
ALTER SEQUENCE orders_id_seq AS bigint;   -- also raises MAXVALUE if it was the integer default

-- Stopgap 1: if max(id) is far below the sequence (rollbacks burned IDs), everything
-- above max(id) is free, so win back the unused top of the range
SELECT max(id) FROM orders;                          -- say it returns 1500000000
ALTER SEQUENCE orders_id_seq RESTART WITH 1500000001;

-- Stopgap 2: if the application tolerates negative IDs, use the integer range's
-- unused negative half (about 2.1 billion more values) while you plan the real fix
ALTER SEQUENCE orders_id_seq MINVALUE -2147483648 RESTART WITH -2147483648;
```

---

## Prevention

```sql
-- Always use bigserial or IDENTITY for new tables:
CREATE TABLE orders (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    ...
);
```

---

## Lab Exercise

```sql
-- Create a sequence with artificially tiny limit
CREATE SEQUENCE tiny_seq AS integer MAXVALUE 5 NO CYCLE;
CREATE TABLE seq_test (id int DEFAULT nextval('tiny_seq') PRIMARY KEY, val text);

-- Insert until overflow
DO $$
DECLARE inserted int := 0;
BEGIN
    FOR i IN 1..10 LOOP
        BEGIN
            INSERT INTO seq_test (val) VALUES ('row_' || i);
            inserted := inserted + 1;
            RAISE NOTICE 'Insert % succeeded, current id=%', i, currval('tiny_seq');
        EXCEPTION WHEN OTHERS THEN
            RAISE NOTICE 'Insert % FAILED: %', i, SQLERRM;
        END;
    END LOOP;
    RAISE NOTICE 'Inserted % rows total', inserted;
END;
$$;

SELECT * FROM seq_test;

-- Fix the sequence: bigint, with the full range
ALTER SEQUENCE tiny_seq AS bigint MAXVALUE 9223372036854775807;
INSERT INTO seq_test (val) VALUES ('after fix');
SELECT * FROM seq_test;

-- Now the second failure mode: the sequence has room, the column doesn't
ALTER SEQUENCE tiny_seq RESTART WITH 2147483648;
INSERT INTO seq_test (val) VALUES ('too big for int');   -- ERROR: integer out of range
ALTER TABLE seq_test ALTER COLUMN id TYPE bigint;
INSERT INTO seq_test (val) VALUES ('fits now');
SELECT * FROM seq_test ORDER BY id DESC LIMIT 2;

-- Clean up
DROP TABLE seq_test;
DROP SEQUENCE tiny_seq;
```

**Write down**: At which insert did it fail? ___  
What command fixed it? ___

# T-08 — Sequence Overflow: "integer out of range"

**Frequency in production**: Moderate — devastating when it hits
**Severity**: P0 — all INSERTs fail to this table
**Related**: Data type fundamentals

---

## The Symptom

```
ERROR:  integer out of range
```

All INSERTs to a specific table fail. Completely.

---

## Why This Happens

`serial` (int) max = 2,147,483,647. Once the sequence hits that, every new INSERT fails.
High-volume tables, tables with many rollbacks (sequence increments even on rollback), or old tables using `serial` instead of `bigserial`.

---

## Diagnosis

```sql
-- Check all sequences and how close to limit
SELECT
    seq.relname                                                         AS sequence,
    pg_sequence.seqtypid::regtype                                       AS type,
    pg_sequence.seqmax                                                   AS max_value,
    last_value,
    round(last_value::numeric / pg_sequence.seqmax * 100, 2)            AS pct_used
FROM pg_class seq
JOIN pg_sequence ON pg_sequence.seqrelid = seq.oid
WHERE seq.relkind = 'S'
ORDER BY pct_used DESC NULLS LAST;

-- Alert: pct_used > 75%
-- Emergency: pct_used > 90%
```

---

## Immediate Fix

```sql
-- If sequence ran out but actual max(id) is much lower (rollbacks consumed IDs):
SELECT max(id) FROM orders;
ALTER SEQUENCE orders_id_seq RESTART WITH [max_id + 1];
ALTER SEQUENCE orders_id_seq AS bigint MAXVALUE 9223372036854775807;

-- If table is large: must change column type too
ALTER TABLE orders ALTER COLUMN id TYPE bigint;
ALTER SEQUENCE orders_id_seq AS bigint;
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

-- Fix: extend sequence to bigint
ALTER SEQUENCE tiny_seq AS bigint MAXVALUE 9223372036854775807;
INSERT INTO seq_test (val) VALUES ('after fix');
SELECT * FROM seq_test;

-- Clean up
DROP TABLE seq_test;
DROP SEQUENCE tiny_seq;
```

**Write down**: At which insert did it fail? ___  
What command fixed it? ___

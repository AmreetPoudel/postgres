# Lab 001 — See the Architecture With Your Own Eyes

**Topic**: 1.1 — PostgreSQL Architecture Overview  
**Read first**: `curriculum/1-internals/1.1-architecture-overview.md`  
**Time**: ~45 minutes  
**Goal**: Every command here proves something from the reading. Don't run blind — know what you're proving before you run it.

---

## Setup Check

Before anything, confirm your PostgreSQL 16 is running:

```bash
pg_lsclusters          # Debian/Ubuntu
# or
systemctl status postgresql-16   # RHEL/Rocky
# or
pg_ctl status -D /var/lib/postgresql/16/main
```

Also export PGDATA so commands are shorter:
```bash
export PGDATA=$(psql -U postgres -tAc "SHOW data_directory;")
echo $PGDATA
```

---

## Task 1: The Process Model (Part 1 + 3 from reading)

**What you're proving**: Every connection = one OS process. Background workers are real processes.

```bash
ps aux --forest | grep -E "postgres|PID" | grep -v grep
```

The `--forest` flag shows parent-child relationships. You should see:
- The postmaster at the top
- All background workers forked from it
- Your current session as one of the children

**Write here — annotate each process you see:**
```
PID     | Process Name           | What it does (from your reading)
--------|------------------------|----------------------------------
        | postmaster             | 
        | bgwriter               |
        | checkpointer           |
        | walwriter              |
        | autovacuum launcher    |
        | stats collector        |
        | your psql session      |
```

**Challenge**: Open a second terminal and connect with psql. Then re-run `ps aux`. What new process appears? What's its PID? What does that prove?

---

## Task 2: Shared Memory (Part 2 from reading)

**What you're proving**: shared_buffers is real allocated RAM, not a config number.

```sql
-- What is shared_buffers configured as?
SHOW shared_buffers;

-- How much shared memory does PostgreSQL actually have?
SELECT name, setting, unit 
FROM pg_settings 
WHERE name IN ('shared_buffers', 'work_mem', 'maintenance_work_mem', 'wal_buffers');
```

On Linux, see the actual shared memory segment:
```bash
ipcs -m | head -5
# SHMID = shared memory ID
# SHMEM size should reflect shared_buffers
```

**Key question**: shared_buffers shows 128MB by default. That feels small. Why? And where does the rest of PostgreSQL's memory go? (Hint: look at work_mem — and think about 200 sessions each doing a sort...)

**Calculate this**:
- `work_mem` default = 4MB
- If 200 sessions each do a sort simultaneously, max RAM for sorts = ?
- Now set work_mem to 64MB. Same calculation. What's the risk?

---

## Task 3: PGDATA — The Filesystem Layout (Part 4 from reading)

**What you're proving**: Tables are files named by OID. Databases are directories.

```bash
# List the base/ directory
ls -la $PGDATA/base/

# Each number is a database OID. Match them:
psql -U postgres -c "SELECT oid, datname FROM pg_database ORDER BY oid;"
```

**Do the OIDs match the directory names?** They should. Write it down:
```
OID   | Directory in base/  | Database name
------|---------------------|---------------
1     | base/1/             | template1
...   | ...                 | ...
```

Now go one level deeper — find your table's file:
```sql
-- Connect to a database that has a table (use your own DB)
\c mydb

-- Find orders table's file location
SELECT 
    relname,
    relfilenode,
    relpages,
    pg_size_pretty(pg_relation_size(oid)) as size
FROM pg_class 
WHERE relname = 'orders';   -- change to any table you have
```

```bash
# Now find that file on disk
DB_OID=$(psql -U postgres -tAc "SELECT oid FROM pg_database WHERE datname='mydb';")
# Use the relfilenode from above
ls -lh $PGDATA/base/$DB_OID/<relfilenode>

# You'll also see two companion files:
# <relfilenode>_fsm   = free space map (where free space is in this file)
# <relfilenode>_vm    = visibility map (which pages have only live tuples)
```

**If you insert a large number of rows, what do you expect to happen to the file size?**
```sql
-- Test it: insert some rows and check the file size before and after
INSERT INTO orders (customer_id, status) 
SELECT generate_series(1,10000), 'pending';

-- Check size again
SELECT pg_size_pretty(pg_relation_size('orders'));
```

---

## Task 4: WAL Segments (Part 5 from reading)

**What you're proving**: WAL is a real file on disk. It grows with writes. The filename encodes position.

```bash
ls -lh $PGDATA/pg_wal/
```

**Observe**:
- How many segments exist?
- What is each file's size? (Should be 16MB each)
- What does the filename look like? (24 hex characters)

Now watch WAL advance in real time:

```sql
-- See current WAL position (LSN = Log Sequence Number)
SELECT pg_current_wal_lsn(), pg_walfile_name(pg_current_wal_lsn());

-- Save this LSN
-- Now do a bunch of writes:
INSERT INTO orders (customer_id, status) 
SELECT generate_series(1,50000), 'pending';

-- Check WAL position again
SELECT pg_current_wal_lsn(), pg_walfile_name(pg_current_wal_lsn());

-- How much WAL was generated by those inserts?
SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), '<paste your first LSN here>');
-- Result is in bytes
```

**What does this tell you about the write amplification of a large INSERT?**

---

## Task 5: MVCC — Dead Tuples With Your Own Eyes (Part 6 from reading)

**What you're proving**: DELETE and UPDATE don't remove data immediately. Dead tuples accumulate.

```sql
-- Create a test table
CREATE TABLE mvcc_test (id serial, data text);
INSERT INTO mvcc_test (data) SELECT md5(random()::text) FROM generate_series(1,10000);

-- Check current state
SELECT 
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    last_autovacuum,
    last_autoanalyze
FROM pg_stat_user_tables 
WHERE relname = 'mvcc_test';
```

Now delete half the rows:
```sql
DELETE FROM mvcc_test WHERE id % 2 = 0;

-- Check dead tuples — run immediately, before autovacuum fires
SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE relname = 'mvcc_test';
```

**What do you see?** n_dead_tup should be ~5000. Those are the deleted rows — still physically on disk.

```bash
# Check the file size — it hasn't shrunk
psql -c "SELECT pg_size_pretty(pg_relation_size('mvcc_test'));"
```

Now run VACUUM manually and check again:
```sql
VACUUM VERBOSE mvcc_test;

-- Check after vacuum
SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE relname = 'mvcc_test';

-- Did the file size shrink?
SELECT pg_size_pretty(pg_relation_size('mvcc_test'));
```

**Key question**: VACUUM ran. Dead tuples are gone. But did the file size shrink? Why or why not? (This is the difference between VACUUM and VACUUM FULL — we'll cover this in 1.7)

---

## Task 6: The Query Lifecycle — Watch the Planner (Part 7 from reading)

**What you're proving**: The planner makes decisions based on statistics. Wrong statistics = wrong plan.

```sql
-- Create a table with intentionally skewed data
CREATE TABLE query_test (id serial, category text, value int);

INSERT INTO query_test (category, value)
SELECT 
    CASE WHEN id % 100 = 0 THEN 'rare' ELSE 'common' END,
    (random() * 1000)::int
FROM generate_series(1,100000) id;

CREATE INDEX ON query_test(category);

-- Run ANALYZE to update statistics
ANALYZE query_test;

-- Now look at two queries and their plans:

-- Query 1: looking for common rows (~99,000 rows)
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM query_test WHERE category = 'common';

-- Query 2: looking for rare rows (~1,000 rows)
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM query_test WHERE category = 'rare';
```

**What do you observe?**
- Did the planner use the index for both queries? Or just one?
- Why would the planner choose a sequential scan for 'common' but an index scan for 'rare'?
- What does "Buffers: shared hit=X read=Y" tell you?

Now deliberately break the statistics:
```sql
-- Insert 1 million 'rare' rows WITHOUT running ANALYZE
INSERT INTO query_test (category, value)
SELECT 'rare', (random() * 1000)::int FROM generate_series(1,1000000);

-- Run the same query — the planner still thinks 'rare' is 1% of data
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM query_test WHERE category = 'rare';
```

**What plan does the planner pick now? Is it optimal? This is exactly how queries go from fast to slow in production.**

---

## Your Findings

Fill this in after completing all tasks:

**Most surprising thing I discovered:**
```
(write here)
```

**Something I don't understand yet:**
```
(write here — these become questions for the next session)
```

**Connection I made between the reading and what I saw:**
```
(write here)
```

---

## Completion Checklist

- [ ] Task 1: Mapped all PostgreSQL processes to their roles
- [ ] Task 2: Calculated work_mem risk at scale
- [ ] Task 3: Matched database OIDs to directories, found a table's physical file
- [ ] Task 4: Watched WAL LSN advance during writes
- [ ] Task 5: Saw dead tuples accumulate and survive after VACUUM (file size didn't shrink)
- [ ] Task 6: Broke query planning with stale statistics
- [ ] Updated `progress.md` with what was done today
- [ ] Added any new questions to `backlog.md`

---

## After This Lab, You Should Know

1. Why connection count is expensive in PostgreSQL
2. What lives in shared memory and why
3. How to find any table's data file on disk
4. What WAL looks like physically and how much a write generates
5. Why deleting rows doesn't shrink files
6. Why stale statistics cause slow queries

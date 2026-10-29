# work_mem — tuning note

**What:** The memory each sort or hash step in a query may use before spilling to disk (4MB default, per operation, not per connection).

**Why:** Too low makes big sorts spill to disk and run slowly, too high lets many sessions together run the server out of RAM.

## How

1. **Find heavy queries:** slow query log, pg_stat_statements.
2. **Check for a spill:**
   ```sql
   EXPLAIN (ANALYZE, BUFFERS) <query>;
   -- Spill:  Sort Method: external merge  Disk: 46000kB
   -- Spill:  Hash ... Batches: 4   (anything above 1)
   ```
   No spill → work_mem isn't the problem. Look at indexes and the plan instead.
3. **Test in your session only:**
   ```sql
   SET work_mem = '128MB';   -- start at ~2× the Disk: value
   EXPLAIN (ANALYZE, BUFFERS) <query>;
   -- Goal:   Sort Method: quicksort  Memory: ...
   ```
4. **Apply narrowly:**
   ```sql
   ALTER ROLE reporting SET work_mem = '128MB';   -- one user only
   ```
5. **Global value only if many queries need it**, and check it fits:
   `max_connections × 3 × work_mem` must fit in RAM left after shared_buffers.

## Monitor

```sql
SELECT datname, temp_files, pg_size_pretty(temp_bytes) FROM pg_stat_database;
```
`log_temp_files = 0` logs every spill with its query.

Practice: Lab 002, Task 3.

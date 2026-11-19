# EXPLAIN and EXPLAIN ANALYZE

**EXPLAIN:** Shows how PostgreSQL plans to run a query and its estimated row counts; does not execute it.

**EXPLAIN ANALYZE:** Executes the query and shows the plan, actual row counts and timings. Compare estimated rows with actual rows.

**For a slow SELECT:** Start with `EXPLAIN SELECT ...;`, then use this when ready to run it:
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
```
`BUFFERS` shows cache hits and reads; a read can come from the Linux page cache, not necessarily disk.

**Risk:** EXPLAIN ANALYZE runs the query again and adds real load; with INSERT, UPDATE or DELETE, it really changes data.

**Different command:** `ANALYZE table_name;` refreshes table statistics for the planner.

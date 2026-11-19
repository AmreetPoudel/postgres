# Indexes

**Why:** An index helps PostgreSQL find selected rows without scanning the whole table; when most rows match, a full scan can be cheaper.

**How:**
1. Run `EXPLAIN (ANALYZE, BUFFERS)` on a slow SELECT.
2. If estimated and actual row counts differ greatly, refresh statistics with `ANALYZE table_name;` and check again.
3. Consider an index on columns used to filter, join or sort; compare the plan and timing before and after.
4. Before removing an index, check usage over a representative workload, including reports and replicas, and whether it supports a constraint.

**Risk:** Extra indexes use disk and add write overhead; an unused-looking index may still enforce a primary key or uniqueness, or serve an occasional report. A sequential scan alone does not mean an index is missing.

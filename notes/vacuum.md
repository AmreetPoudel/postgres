# VACUUM (3TB production server)

**Why:** UPDATE and DELETE leave dead rows behind. VACUUM makes their space reusable, but plain VACUUM doesn't shrink files.

**How:**
1. Find the tables with the most dead rows:
   ```sql
   SELECT relname, n_dead_tup, last_autovacuum
   FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 10;
   ```
2. Vacuum only those, at a quiet time: `VACUUM (VERBOSE, ANALYZE) big_table;`
3. Check `0 are dead but not yet removable` in the output. Above 0 means an open transaction is blocking cleanup.

**Risk:**
- Don't run `VACUUM FULL` on production: it locks the table completely until it finishes, which is hours on a big table. Use `pg_repack` if you must shrink.
- A database-wide VACUUM on 3TB means heavy I/O for hours. Vacuum table by table, and leave routine work to autovacuum.

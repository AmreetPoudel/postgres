# work_mem

**Why:** Each sort or hash in a query gets this much memory (4MB default). Too low, big queries spill to disk and get slow. Too high, many sessions together can run the server out of RAM.

**How:**
1. Run `EXPLAIN (ANALYZE) <slow query>`.
2. If you see `Sort Method: external merge Disk`, it spilled. No spill, work_mem isn't the problem.
3. Test: `SET work_mem = '128MB';` and run EXPLAIN again. Want `Sort Method: quicksort Memory`.
4. Apply only where needed: `ALTER ROLE reporting SET work_mem = '128MB';`

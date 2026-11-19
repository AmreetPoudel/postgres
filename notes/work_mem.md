# work_mem

**Why:** Each sort or hash in a query gets this much memory (4MB default). Too low, big queries spill to disk and get slow. Too high, many sessions together can run the server out of RAM.

**Remember:** An idle connection reserves no work_mem; each operation allocates memory as needed. Hash operations can use 2 × work_mem by default in PG16, and parallel workers add memory use too.

**How:**
1. Run `EXPLAIN (ANALYZE) <slow query>`.
2. If you see `Sort Method: external merge Disk`, it spilled. No spill, work_mem isn't the problem.
3. Test: `SET work_mem = '128MB';` and run EXPLAIN again. Want `Sort Method: quicksort Memory`.
4. Apply only where needed: `ALTER ROLE reporting SET work_mem = '128MB';`

**Risk:** Example budget = simultaneously active connections × sorts per query × work_mem.
- 600 active connections × 3 sorts × 4MB ≈ 7GiB
- 600 active connections × 3 sorts × 128MB = 225GiB, before shared_buffers and other memory.
Even a per-role increase needs a concurrency budget.

**Future tuning script:** At the end of the course, calculate a starting configuration using CPU, RAM, storage and measured workload/concurrency; back up the config, show proposed changes, then write approved values to a config file and identify reload/restart requirements. CPU and RAM alone cannot determine a safe work_mem.

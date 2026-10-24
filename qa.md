# PostgreSQL DBA Interview Q&A Bank

> Public, growing Q&A bank covering PostgreSQL internals, administration, tuning, replication, HA, and backup.
> All questions arose from real study sessions, lab work and content reviews.
> **Format**: Question → Answer → (Where relevant) Common Trap / Gotcha

---

## Category Index

- [Architecture](#architecture)
- [Process Model](#process-model)
- [Memory](#memory)
- [Storage](#storage)
- [WAL (Write-Ahead Logging)](#wal)
- [MVCC](#mvcc)
- [VACUUM & Bloat](#vacuum--bloat)
- [Query Planning & Execution](#query-planning--execution)
- [The Three Resource Bottlenecks](#the-three-resource-bottlenecks)
- [Indexes](#indexes)
- [Locks & Concurrency](#locks--concurrency)
- [Configuration](#configuration)
- [Backup & Recovery](#backup--recovery)
- [Replication](#replication)
- [High Availability](#high-availability)
- [Security (pg_hba, SSL)](#security)
- [Performance & Benchmarking](#performance--benchmarking)

---

## Architecture

**Q1: What are the main components of PostgreSQL's process architecture?**

> **Answer**: PostgreSQL uses a **multi-process model** (not multi-threaded). Key components:
> - **Postmaster**: supervisor process, listens on port 5432, forks a new backend per client connection
> - **Backend processes**: one per client session — handles parse, plan, execute
> - **Background processes**: checkpointer, background writer, WAL writer, autovacuum launcher and workers, logical replication launcher, WAL archiver (if archiving is on), one WAL sender per standby, and on a standby the WAL receiver and the startup process. There has been no stats collector process since PostgreSQL 15
> - **Shared memory**: all backends share this — contains shared_buffers, WAL buffers, lock table, proc array
>
> **Common trap**: PostgreSQL is NOT multi-threaded. Each connection = 1 OS process. 500 connections = 500 processes = ~5GB RAM overhead before any queries run.

---

**Q2: What is the postmaster and what happens if it dies?**

> **Answer**: The postmaster is the root supervisor process. It listens on port 5432, forks backends for each client, initializes shared memory on startup.
>
> If the postmaster dies → all client connections terminate immediately. Database must be restarted by an external tool (systemd, Patroni). This is why HA clusters exist — to restart or failover automatically.

---

**Q3: When you run `psql -U amrit -d mydb`, what OS process handles your session?**

> **Answer**: The postmaster receives the TCP connection on port 5432 and calls `fork()` — a Linux syscall that clones itself. The clone becomes your **dedicated backend process**. The original postmaster keeps listening for new connections.
>
> Your session = exactly one OS process. If your query fails with an error, only your session is affected. But if the process *crashes* (a segfault, or killed with SIGKILL), the postmaster resets every session and runs crash recovery. See the backend-crash question below.
>
> **Common wrong answer**: "PostgreSQL handles it with threads." Wrong. Full OS processes. See `ps aux --forest | grep postgres` to verify.

---

**Q4: Why does PostgreSQL use processes instead of threads?**

> **Answer**: Mostly **robustness and history**. Each backend's private memory is its own, so a bug can't silently scribble over another session's private state, and when a backend crashes, the postmaster survives to clean up. It does *not* let everyone carry on, though: shared memory might be damaged, so it disconnects every session and runs crash recovery, automatically and usually within seconds. In a multi-threaded server the same crash would take down the whole process until something outside restarted it.
>
> **The tradeoff**: Process startup is slower, and each process costs ~5-10MB RAM. At 1000 connections, that's 5-10GB RAM just for process overhead — before any query runs. This is why connection pooling is critical.
>
> **Historical reason**: When PostgreSQL was designed in the early 1990s, threading libraries were not reliably portable across Unix systems. Processes were the safe choice. The architecture has stayed because the isolation benefits are real.

---

**Q5: You have 1000 clients connecting directly to PostgreSQL. What are the two main problems?**

> **Answer**:
> 1. **RAM exhaustion**: 1000 processes × ~10MB each = 10GB RAM just for connection overhead, before queries run
> 2. **Context-switch CPU overhead**: The OS scheduler has to context-switch between 1000 processes. At high connection counts, CPUs spend more time switching than doing actual work.
>
> **Fix**: PgBouncer in front of PostgreSQL. PgBouncer maintains a small pool of actual PostgreSQL connections (e.g., 100) and queues/multiplexes the 1000 client connections across them.

---

## Process Model

**Q6: Name all PostgreSQL background workers and what each one does.**

> **Answer**:
> | Process | Job |
> |---------|-----|
> | WAL Writer | Writes WAL buffers to `pg_wal/` in the background and flushes asynchronous commits (a synchronous COMMIT is flushed by the committing backend itself) |
> | Checkpointer | Periodically flushes all dirty pages from shared_buffers to data files on disk |
> | Background Writer (bgwriter) | Gradually pre-flushes dirty pages between checkpoints to smooth I/O |
> | Autovacuum Launcher | Monitors tables, spawns autovacuum workers when tables need cleaning |
> | Autovacuum Worker | Reclaims dead tuples, updates statistics, prevents transaction ID wraparound |
> | Logical replication launcher | Starts apply workers for logical replication subscriptions |
> | WAL Archiver | Copies completed WAL segments to archive location (if archiving enabled) |
> | WAL Receiver | (Standby only) Connects to the primary's WAL sender and writes the streamed WAL to local `pg_wal/` |
> | Startup process | (Standby, or crash recovery) Replays WAL into the data files |
>
> No stats collector: since PostgreSQL 15 the `pg_stat_*` counters live in shared memory.
>
> **What breaks when bgwriter is misconfigured**: If bgwriter is too slow, the checkpointer has to do huge bursts of I/O → checkpoint spikes → query latency spikes during checkpoint.

---

**Q7: What is the difference between the bgwriter and the checkpointer?**

> **Answer**: Both write dirty pages to disk, but at different times and for different reasons:
> - **bgwriter**: Runs continuously, writing dirty pages *between* checkpoints to spread I/O load evenly over time
> - **Checkpointer**: Runs at checkpoint time, ensuring ALL dirty pages are flushed to disk so crash recovery only needs to replay WAL from that checkpoint forward
>
> Think of bgwriter as doing dishes throughout the day, and checkpointer as making sure every dish is clean before closing the kitchen.

---

**Q8: A backend process is killed by the OOM killer (SIGKILL). What happens to the other 200 sessions?**

> **Answer**: They're all disconnected. The postmaster treats any backend dying abnormally as a crash: shared memory might be corrupt, so it terminates every other session and runs crash recovery, then accepts connections again, usually within seconds.
> ```
> LOG:  server process (PID 1890) was terminated by signal 9: Killed
> LOG:  terminating any other active server processes
> LOG:  all server processes terminated; reinitializing
> ```
> This is why you never `kill -9` a backend to stop a query (use `pg_cancel_backend()` or `pg_terminate_backend()`), and why OOM kills are full outages. Prevention: sane `work_mem`, `vm.overcommit_memory = 2`, or at least protecting the postmaster's OOM score.
>
> **Common wrong answer**: "Only that session dies, the others are isolated." Their *private* memory is isolated; the server-wide reset still hits everyone.

---

**Q9: You run `ps` on a PostgreSQL 16 server and can't find the stats collector. Why?**

> **Answer**: It no longer exists. PostgreSQL 15 replaced it: each backend accumulates its own counters and flushes them into shared memory, where the `pg_stat_*` views read them. They're written to `pg_stat/` at a clean shutdown, and discarded after a crash or an immediate shutdown, so expect counters to reset after crash recovery.
>
> What you *will* see in `ps`: checkpointer, background writer, walwriter, autovacuum launcher, logical replication launcher, and on standbys the startup process and walreceiver.

---

## Memory

**Q10: What lives in PostgreSQL's shared memory?**

> **Answer**:
> - **shared_buffers**: The buffer pool — cached 8KB pages of table/index data. All backends read/write here. Main performance lever.
> - **WAL buffers**: WAL records are written here before the WAL writer flushes to disk
> - **Lock table**: Tracks all held locks across all backends
> - **Proc array**: One slot per backend, used by MVCC to compute transaction visibility snapshots
> - **Buffer descriptors**: Metadata for each buffer in shared_buffers (which file/page, dirty bit, pin count)

---

**Q11: `shared_buffers` is set to 8GB. You have 200 sessions each doing a sort with `work_mem = 64MB`. What's the maximum theoretical RAM usage?**

> **Answer**: `work_mem` is per-sort-operation, not per-session.
> A single session doing a query with 3 sort nodes can use `3 × work_mem`.
> With 200 sessions each having 3 sort operations: `200 × 3 × 64MB = 38.4GB` — plus 8GB shared_buffers, plus OS, plus other processes.
>
> **This is how you OOM a server.** `work_mem` default is 4MB for a reason. Set it globally only as low as you need. For specific heavy queries, use `SET work_mem = '256MB'` within that session only.
>
> **Rule of thumb**: `work_mem = (RAM - shared_buffers) / (max_connections × 3)`
>
> Hash operations may use up to `work_mem × hash_mem_multiplier` (2× by default since PG15), and each parallel worker gets its own allowance.

---

**Q12: What is the PostgreSQL buffer pool and why can't we just rely on the OS page cache?**

> **Answer**: The buffer pool (`shared_buffers`) is PostgreSQL's own RAM cache for data pages. PostgreSQL manages it directly rather than relying solely on the OS page cache because:
> 1. **Control**: PostgreSQL can implement its own eviction policy (clock-sweep), tuned for database access patterns
> 2. **Dirty page tracking**: PostgreSQL needs to know exactly which pages are dirty to manage checkpoints correctly
> 3. **Shared access**: All backend processes can access shared_buffers simultaneously with proper synchronization
>
> **Why not set shared_buffers = all RAM?** Because the OS page cache also caches data. Setting shared_buffers too high starves the OS cache, which PostgreSQL also uses for data not in shared_buffers. Typical setting: 25% of RAM for shared_buffers, let OS cache handle the rest.

---

**Q13: What is the fundamental difference between PostgreSQL's Private Memory and Shared Memory?**

> **Answer**:
> - **Shared Memory**: Allocated once by Postmaster during server startup using OS shared memory (POSIX/SysV). Accessible by ALL backend processes and background workers simultaneously. Contains `shared_buffers` (cached 8KB table/index pages), the Lock Table (concurrency control), and WAL buffers.
> - **Private Memory**: Allocated dynamically by each individual backend process for its own session/query. Completely isolated from all other processes; when the backend exits or disconnects, the OS automatically reclaims 100% of it. Contains query parse trees, execution plans, and `work_mem` (used for sorting and hashing).
>
> **The Production Gotcha**: If a query has 3 sort nodes and runs with `work_mem = 64MB`, a single backend process can consume `3 × 64MB = 192MB` of private RAM. With 100 concurrent connections doing that, the server needs ~19GB of RAM *just for private memory*, which can trigger the OS OOM (Out Of Memory) Killer to kill PostgreSQL.

---

**Q14: Does PostgreSQL ever process query data directly on disk, or always through memory? What is a Cache Hit vs Cache Miss?**

> **Answer**:
> In standard query execution, **PostgreSQL ALWAYS serves data from memory (`shared_buffers`; temporary tables use backend-local buffers instead)**. The CPU cannot execute SQL operations or filter rows directly on disk blocks.
>
> - **Cache Hit**: The requested 8KB page is already sitting in one of the 8KB slots in `shared_buffers`. The backend reads it directly from RAM in nanoseconds/microseconds without touching the disk.
> - **Cache Miss**: The requested 8KB page is not in `shared_buffers`. PostgreSQL must issue a read call to load that 8KB page from the storage drive into a slot in `shared_buffers` first, and only then serves the rows to the client.
>
> *(Exception: Massive table scans or `VACUUM` use a tiny temporary "ring buffer" of 256KB-16MB to avoid blowing away the main cache, but it still loads 8KB pages into RAM before reading).*
>
> **Production Benchmark Metric**: In production OLTP environments, you monitor the **Buffer Cache Hit Ratio** via `pg_statio_user_tables`:
> `sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) * 100%`
> In a healthy production database, this should consistently stay above **99%**. If it dips below 95%, your disk I/O spikes and query latency degrades.

---

**Q15: What is a "dirty page" in PostgreSQL? How does it differ from a "clean page"?**

> **Answer**:
> - **Clean Page**: An 8KB page in `shared_buffers` whose content is identical bit-for-bit to the data file on the SSD. If PostgreSQL needs room for new data, it can evict a clean page instantly without doing any disk write.
> - **Dirty Page**: An 8KB page in `shared_buffers` that has been modified by an `INSERT`, `UPDATE`, or `DELETE`, but has **not yet been written to the table file on disk**.
>
> **Lifecycle of a Dirty Page**:
> 1. A backend process modifies rows inside the 8KB page in `shared_buffers` and marks the page's dirty flag as `true`.
> 2. The backend writes the transaction's changes to the WAL buffer and commits.
> 3. The page remains "dirty" in RAM until the **Checkpointer** or **Background Writer (bgwriter)** sweeps through, writes the 8KB page to the SSD, and clears the dirty flag back to `false` (making it clean again).
>
> **DBA Implication**: You can observe the volume of dirty buffers via the `pg_buffercache` extension. A massive surge in dirty pages means high upcoming write pressure on the storage subsystem during the next checkpoint.

---

## Storage

**Q16: Where on disk does PostgreSQL store a table's data? Walk through the path.**

> **Answer**: Every object in PostgreSQL has an OID (Object Identifier). Tables are stored at:
> `$PGDATA/base/<database_oid>/<table_relfilenode>`
>
> To find it:
> ```sql
> SELECT oid FROM pg_database WHERE datname = 'mydb';       -- e.g., 16384
> SELECT relfilenode FROM pg_class WHERE relname = 'orders'; -- e.g., 24601
> -- File is at: $PGDATA/base/16384/24601
> ```
>
> Each table also has companion files:
> - `24601_fsm` — Free Space Map (where free space is within pages)
> - `24601_vm` — Visibility Map (which pages are all-visible or all-frozen, for index-only scans and VACUUM optimization)
>
> If a table grows beyond 1GB, PostgreSQL splits it: `24601`, `24601.1`, `24601.2`, etc.
>
> Shortcut: `SELECT pg_relation_filepath('orders');` returns the path relative to `$PGDATA`.

---

**Q17: What is an 8KB page in PostgreSQL?**

> **Answer**: PostgreSQL reads and writes data in fixed-size blocks called **pages** (8KB by default). Every table file is a sequence of pages. The page layout:
> - **Page header** (24 bytes): LSN of last WAL record touching this page, checksum, free space info
> - **Item pointers**: Array of (offset, length) pairs pointing to where each row starts
> - **Free space**: In the middle, shrinks as rows are added
> - **Row data (tuples)**: Stored from the bottom up. Each tuple has: `xmin`, `xmax`, null bitmap, then actual column data
>
> **Why 8KB?** A long-standing compile-time default (`BLCKSZ`), balancing I/O efficiency against wasted space; it's twice the usual 4KB OS memory page. Tunable at compile time but almost never changed.

---

## WAL

**Q18: What is WAL and what problem does it solve?**

> **Answer**: WAL = Write-Ahead Log. It's a sequential log of every change to the database, stored in `$PGDATA/pg_wal/`.
>
> **The problem it solves**: If PostgreSQL writes data directly to table files and the server crashes mid-write, the file is corrupt — some pages written, some not. There's no way to know which data to trust.
>
> **The WAL rule**: a changed page may not reach its data file until the WAL record describing the change has been flushed to disk, and a commit isn't acknowledged until its WAL is flushed. If the server crashes, PostgreSQL reads WAL from the last checkpoint and replays every record, committed or not; transactions without a commit record simply count as aborted. Data files become consistent.
>
> **Three uses of WAL**:
> 1. Crash recovery (replay WAL after unclean shutdown)
> 2. Point-in-time recovery (replay WAL to any past moment)
> 3. Streaming replication (standby replays primary's WAL in real-time)

---

**Q19: What happens if you delete `pg_wal/` while PostgreSQL is running?**

> **Answer**: Much worse than "losing a few commits":
> 1. PostgreSQL panics immediately with `PANIC: could not write to file "pg_wal/..."` and crashes
> 2. On restart, crash recovery is **impossible** — PostgreSQL cannot bring data files to a consistent state without WAL
> 3. The database may not start at all, or starts in a corrupt state
> 4. If you have replication, the standby loses its WAL stream and falls behind permanently
>
> **Common wrong answer**: "We lose a few commits." No — we potentially lose the entire ability to start the database.

---

**Q20: What is an LSN (Log Sequence Number)?**

> **Answer**: An LSN is a 64-bit pointer into the WAL stream, written as `X/XXXXXXXX` (e.g., `0/15D3A00`). It represents a byte offset within the WAL.
>
> - Every WAL record has an LSN
> - Every data page stores the LSN of the last WAL record that modified it
> - Checkpoints record an LSN — recovery starts from here
> - Replication lag is measured in LSN difference between primary and standby
>
> ```sql
> SELECT pg_current_wal_lsn();               -- current WAL position
> SELECT pg_walfile_name(pg_current_wal_lsn()); -- which segment file
> SELECT pg_wal_lsn_diff('0/16000000', '0/15D3A00'); -- bytes between two LSNs
> ```

---

**Q21: What are WAL segments? How are they named?**

> **Answer**: WAL is stored in 16MB files called **segments** in `pg_wal/`. The filename is 24 hex characters encoding three 8-character fields:
>
> `000000010000000000000001`
> - `00000001` = timeline ID (1 = original, increments after each failover/recovery)
> - `00000000` = high 32 bits of the segment number
> - `00000001` = low 32 bits of the segment number
>
> When a segment fills, a new one starts. Old segments are archived first (if `archive_mode = on`), then recycled or removed after a checkpoint.
>
> **Danger**: If archiving is on but the archiver is failing, WAL segments accumulate in `pg_wal/` until disk is full → database stops accepting writes.

---

**Q22: Who actually flushes WAL to disk when you COMMIT? The WAL writer?**

> **Answer**: No: the committing backend does it itself. With `synchronous_commit = on`, COMMIT writes a commit record and calls `fsync` on WAL up to that point before returning. When many sessions commit at the same moment, one flush covers them all (group commit).
>
> The WAL writer writes WAL buffers out in the background, so backends rarely have to, and it's what flushes *asynchronous* commits. With `synchronous_commit = off`, COMMIT returns before the flush, and a crash can lose up to about 3 × `wal_writer_delay` (600ms by default) of commits, without corrupting anything.

---

## MVCC

**Q23: What does MVCC stand for and why does PostgreSQL use it?**

> **Answer**: **Multi-Version Concurrency Control**.
>
> **Why**: To avoid readers blocking writers and writers blocking readers. In a pure locking model, a long-running UPDATE on a row blocks all SELECTs on that row. At scale, this is a throughput disaster.
>
> **How MVCC works**: Instead of modifying a row in place, PostgreSQL writes a **new version** of the row. The old version stays on disk. Each query sees a **snapshot** of the database: in READ COMMITTED (the default) a fresh one for every statement, in REPEATABLE READ and SERIALIZABLE one for the whole transaction, taken at its first statement. Old versions are invisible to snapshots taken after they were deleted/updated.
>
> **Common wrong answer**: "MVCC is for recovery from accidental deletion." No. That's what backups and PITR are for. MVCC is purely about concurrency.

---

**Q24: What are `xmin` and `xmax` on a PostgreSQL row?**

> **Answer**: Every row (tuple) in PostgreSQL carries two hidden fields:
> - `xmin`: Transaction ID that **created** this row version (INSERT or UPDATE that created it)
> - `xmax`: Transaction ID that **ended** this row version (DELETE or the UPDATE that replaced it). Zero if it was never deleted. A non-zero `xmax` can also come from a row lock (`SELECT ... FOR UPDATE`) or a rolled-back DELETE, and those rows are still live.
>
> When a backend runs a query, it computes a **snapshot**: the set of transaction IDs it can see. A row is visible if:
> - `xmin` committed before the snapshot was taken
> - AND `xmax` is 0, or aborted, or hadn't committed when the snapshot was taken
>
> This is how PostgreSQL serves different row versions to different transactions simultaneously without locks.
>
> ```sql
> -- See xmin and xmax directly:
> SELECT xmin, xmax, * FROM orders WHERE id = 1;
> ```

---

**Q25: You UPDATE 1 million rows. How many row versions exist immediately after?**

> **Answer**: **2 million** — one old version (xmax set, marked deleted) and one new version (xmin set) for each of the 1 million rows. Both physically exist on disk until VACUUM cleans the old versions.
>
> This is why UPDATE-heavy workloads cause table bloat and why autovacuum needs to keep up with write rate.

---

**Q26: A session has been `idle in transaction` for 3 hours. Is it blocking VACUUM?**

> **Answer**: It depends on what it did, so check rather than guess:
> ```sql
> SELECT pid, state, backend_xid, backend_xmin, age(backend_xmin) AS snapshot_age
> FROM pg_stat_activity WHERE state = 'idle in transaction';
> ```
> - `backend_xid` set: it wrote something, its transaction counts as still running, and VACUUM can't remove anything that died after it began — in every table of the database
> - `backend_xmin` set: it holds a snapshot (REPEATABLE READ, or an open cursor): same effect
> - Both NULL: a read-only READ COMMITTED transaction keeps no snapshot between statements, so it doesn't block VACUUM. It still holds its locks, though, and a lock is enough to block DDL
>
> **Gotcha**: `ORDER BY backend_xmin` or `min(backend_xmin)` fails, because the `xid` type has no ordering. Sort by `age(backend_xmin)` instead.

---

## VACUUM & Bloat

**Q27: What is table bloat and what causes it?**

> **Answer**: Bloat = table file size is much larger than the actual live data.
>
> **Cause**: MVCC keeps old row versions (dead tuples) on disk after DELETE or UPDATE. The table file grows. Even if you delete 90% of your rows, the file doesn't shrink.
>
> **VACUUM** marks dead tuple space as reusable for new inserts. The file size stays the same but space is reused.
>
> **VACUUM FULL** rewrites the entire table to a new file, shrinking it. But it holds an `ACCESS EXCLUSIVE` lock — no reads or writes during VACUUM FULL. Use sparingly on production.
>
> **pg_repack** is the production alternative to VACUUM FULL — rewrites table online with minimal locking.

---

**Q28: What happens if VACUUM never runs?**

> **Answer**: Three cascading problems:
> 1. **Bloat**: Table files grow without bound. Queries scan more pages, get slower.
> 2. **Index bloat**: Indexes also accumulate dead entries. Same slowdown.
> 3. **Transaction ID wraparound** (catastrophic): PostgreSQL uses 32-bit transaction IDs. After ~2.1 billion transactions, they wrap around. Without VACUUM freezing old rows and advancing `relfrozenxid`, PostgreSQL stops assigning new transaction IDs when only 3 million remain, so every write fails until VACUUM catches up (read-only queries keep working). This is an emergency.
>
> **Warning signs**: `age(datfrozenxid)` above ~500 million means autovacuum is falling behind (it starts forced freezing at 200 million). PostgreSQL itself only starts warning 40 million XIDs before the limit.

---

**Q29: Writes fail with "database is not accepting commands to avoid wraparound data loss". What do you do, in order?**

> **Answer**: The PostgreSQL 16 procedure:
> 1. Resolve old prepared transactions (`pg_prepared_xacts`)
> 2. End long-running transactions (large `age(backend_xid)` or `age(backend_xmin)`)
> 3. Drop stale replication slots (large `age(xmin)` or `age(catalog_xmin)`)
> 4. Run a plain `VACUUM` on the database, as a superuser
>
> Read-only queries and VACUUM keep working the whole time.
>
> **Common wrong answers**: "Stop the postmaster and vacuum in single-user mode" (the error's own HINT says so, but the docs say it's no longer necessary or desirable), "`VACUUM FREEZE` everything" (more work than needed), "`VACUUM FULL`" (it needs a new transaction ID, which is exactly what you can't get).

---

## Query Planning & Execution

**Q30: What are the 6 stages of a PostgreSQL query?**

> **Answer**:
> 1. **Parser**: Tokenizes and parses SQL into an Abstract Syntax Tree. Syntax errors caught here.
> 2. **Analyzer/Rewriter**: Resolves table/column names to OIDs, expands views, applies RLS rules, and records which permissions are needed (the executor checks them as execution starts).
> 3. **Planner/Optimizer**: Generates possible execution plans, estimates cost using statistics from `pg_statistic`, picks the cheapest plan.
> 4. **Executor**: Runs the chosen plan — reads pages from shared_buffers or disk, applies filters, performs joins and sorts.
> 5. **Result Formatting**: Converts internal data types to wire protocol format.
> 6. **Client receives rows**: Streamed incrementally (not all at once).

---

**Q31: A query ran in 5ms yesterday. Same query runs in 8 seconds today. Data hasn't changed. What's your first suspect?**

> **Answer**: **Stale statistics → bad query plan**.
>
> If a large batch INSERT, DELETE, or UPDATE happened and ANALYZE wasn't run, `pg_statistic` is stale. The planner estimates the wrong number of rows, picks the wrong plan (e.g., seq scan instead of index scan, or nested loop instead of hash join).
>
> **Diagnosis**:
> ```sql
> EXPLAIN (ANALYZE, BUFFERS) <your query>;
> -- Compare "rows=X" (estimate) vs "rows=X" (actual)
> -- Large discrepancy = stale statistics
>
> -- When were stats last updated?
> SELECT relname, last_analyze, last_autoanalyze
> FROM pg_stat_user_tables WHERE relname = 'orders';
> ```
>
> **Fix**: `ANALYZE orders;` then re-run EXPLAIN to confirm plan changed.
>
> **Other suspects** (if statistics are fresh): autovacuum bloat, index corruption, connection pool exhaustion causing queuing, lock contention.

---

**Q32: What does `EXPLAIN (ANALYZE, BUFFERS)` show and why does it matter?**

> **Answer**:
> - **EXPLAIN**: Shows the plan the planner *would* use (estimates only, query NOT executed)
> - **EXPLAIN ANALYZE**: Actually runs the query, shows estimated vs actual row counts and timing
> - **BUFFERS**: Shows how many 8KB pages were read from shared_buffers (hit) vs from disk (read)
>
> Key things to look for:
> - `rows=100` estimate vs `rows=50000` actual → bad statistics → wrong plan
> - `Buffers: shared hit=5 read=10000` → 10000 pages from disk → cache miss → possible shared_buffers too small or cold data
> - `Sort Method: external merge Disk` → sort spilled to disk → work_mem too low
> - `Seq Scan` on a large table → missing index or planner chose not to use it (check why)

---

## The Three Resource Bottlenecks

**Q33: What are the three ways PostgreSQL performance degrades, and how do you identify each?**

> **Answer**:
>
> **1. DISK** (most common):
> - Symptoms: high `await` in `iostat`, slow commits, checkpoint warnings in logs
> - Check: `iostat -x 1`, `pg_stat_bgwriter` (`checkpoints_req >> checkpoints_timed`)
> - Root causes: undersized IOPS (HDD vs NVMe), wrong checkpoint config, autovacuum I/O storm
>
> **2. MEMORY**:
> - Symptoms: swap in use (`vmstat` si/so nonzero), OOM kills, buffer hit rate < 99%, queries spilling to disk
> - Check: `free -h`, `vmstat 1`, `SELECT blks_hit/(blks_hit+blks_read) FROM pg_stat_database`
> - Root causes: shared_buffers too small, work_mem too high × too many connections
>
> **3. CPU** (least common, often a symptom of the above):
> - Symptoms: high load average, slow query planning, missing indexes
> - Check: `top`, `pg_stat_statements` sorted by `total_exec_time`
> - Root causes: seq scans on large tables, stale statistics causing bad plans, too many connections causing context-switch overhead

---

## Indexes

*(Questions added as sessions progress)*

---

## Locks & Concurrency

**Q34: An `ALTER TABLE` is stuck, and now every plain SELECT on that table hangs too. How do you find the culprit?**

> **Answer**: It's a lock queue. Some session (often `idle in transaction`) holds an ordinary lock on the table; the ALTER waits for ACCESS EXCLUSIVE; and every new query queues behind the ALTER, because PostgreSQL doesn't let later lock requests jump the queue. Find the session that blocks others but isn't waiting itself:
> ```sql
> SELECT pid, state, now() - xact_start AS xact_age, left(query, 60) AS query
> FROM pg_stat_activity
> WHERE pid IN (SELECT unnest(pg_blocking_pids(pid)) FROM pg_stat_activity)
>   AND cardinality(pg_blocking_pids(pid)) = 0;
> ```
> Terminate *that* one, not the waiters. Prevention: run DDL with `SET lock_timeout = '3s'` and retry.
>
> **Gotcha**: the classic "join pg_locks to itself on transactionid" query returns nothing here, because these are table locks. `pg_blocking_pids()` understands every lock type.

---

## Configuration

*(Questions added as sessions progress)*

---

## Backup & Recovery

**Q35: During PITR you overshot the target by 5 minutes. Can you set an earlier `recovery_target_time` and restart?**

> **Answer**: No. WAL can't be un-applied: you have to restore the base backup again and replay to the earlier target. (Moving the target *later* does work: recovery just continues forward.)
>
> That's why the default `recovery_target_action` is `pause`: recovery stops at the target, read-only, so you can check the data before `SELECT pg_wal_replay_resume();` promotes it. Pausing needs `hot_standby = on` (the default); with it off, `pause` acts like `shutdown`.

---

## Replication

**Q36: Your only synchronous standby dies. What happens to commits on the primary, and how do you get writes flowing again?**

> **Answer**: Every COMMIT waits, with no timeout, until a synchronous standby confirms. The change is already durable locally; only the acknowledgement waits. Cancelling the wait doesn't undo anything: the client gets `WARNING: canceling wait for synchronous replication due to user request` and `The transaction has already committed locally, but might not have been replicated to the standby.`
>
> To unblock: change `synchronous_standby_names` (for example to `''`) and reload. To avoid it: `ANY 1 (standby1, standby2)` with two standbys, or let Patroni's `synchronous_mode` manage it. Non-strict mode then drops to async automatically; `synchronous_mode_strict` keeps blocking, by design.
>
> **Common wrong answer**: "Commits wait for `wal_sender_timeout`, then continue." That timeout only closes the connection to an unresponsive standby.

---

## High Availability

**Q37: In a Patroni cluster you run `pg_ctl stop -m immediate` on the primary to test failover. Nothing fails over. Why?**

> **Answer**: Patroni on that node is still alive and still holds the leader lock, so it just starts PostgreSQL again. That's correct: a crashed PostgreSQL on a healthy node is a restart, not a failover.
>
> To test a real failover, crash the whole node (power it off, or kill Patroni and PostgreSQL together). For a planned move, use `patronictl switchover`. The dangerous case is the opposite one: Patroni dies but PostgreSQL keeps running as primary. Nothing demotes it when its lease expires, which is what Patroni's watchdog support protects against.

---

**Q38: You have exactly two data centers. Can Patroni + etcd fail over automatically no matter which site is lost?**

> **Answer**: No. etcd needs a majority, and with two sites one of them always holds the majority of members. Lose that site and the other can't elect a leader, so failover is manual. To survive losing *either* site, put an etcd tie-breaker in a third location (even a small VM), for example one member per site.
>
> The replication setup matters too. Under Patroni's `synchronous_mode`, only the synchronous standby may promote itself automatically. If the synchronous standby was in the lost site, the surviving site's standby won't promote on its own, because it may be missing acknowledged commits. You decide whether to accept that loss.

---

## Security

**Q39: You change pg_hba.conf from `md5` to `scram-sha-256`, and some users can no longer log in. Why?**

> **Answer**: Their stored password is still an md5 hash. A `scram-sha-256` rule can only verify SCRAM-stored passwords, while an `md5` rule accepts both kinds. Safe order: set `password_encryption = scram-sha-256`, keep `md5` rules while every user sets their password again, check that none remain (`SELECT usename, left(passwd, 3) FROM pg_shadow;` shows `md5` vs `SCR`), then switch the rules.

---

## Performance & Benchmarking

*(Questions added as sessions progress)*

---

*Last updated: 2026-10-24 — accuracy review against PostgreSQL 16*
*Questions: 39 | Target: 200+*
*Topics covered: Architecture, Process Model, Memory, Storage, WAL, MVCC, VACUUM & Bloat, Query Planning, Resource Bottlenecks, Locks, Backup & Recovery, Replication, High Availability, Security*

# Progress Log

Format: `[Date] | [Phase.Topic] | [Duration] | [Status]`

---

## Sessions

### Session 001 — 2026-09-15
- **Topic**: Setup & Phase 1.1 Architecture Deep Dive
- **Duration**: ~60 min
- **Status**: 🟡 In Progress (Concepts covered, Q&A refined to 28 questions)
- **Concepts Discussed & Mastered**:
  - Postmaster listening on 5432 & `fork()` connection handoff
  - Multi-process architecture vs multi-threading (fault isolation & blast radius)
  - Private Memory (`work_mem`, parse tree) vs Shared Memory (`shared_buffers`, lock tables)
  - Storage slicing into 8 KB Pages & Cache Hit vs Cache Miss
  - Why `shared_buffers` is 25% (Linux OS Page Cache double buffering, work_mem safety)
  - Dirty Pages vs Clean Pages in memory
  - MVCC basics (UPDATE = insert new row + mark old row deleted with xmax)
  - WAL (Write-Ahead Logging) & Crash Recovery (Startup process replaying REDO log)
- **Next**: User handwriting notes & revising. Next session: Lab 001 hands-on terminal exploration (inspecting real PIDs, shared memory, and 8KB pages).

### Maintenance — 2026-10-24: Content review against PostgreSQL 16
- **Not a study session**: every curriculum, lab and Q&A file was checked against PostgreSQL 16, and every SQL snippet was run on a PG16 server
- **Corrections to Session 001 material** (update the handwritten notes):
  - A crashed backend does NOT leave the other sessions running: the postmaster resets every session and runs crash recovery
  - There is no stats collector process since PostgreSQL 15; statistics live in shared memory
  - At COMMIT, the committing backend flushes WAL itself; the WAL writer handles background writes and async commits
  - Crash recovery replays all WAL, committed or not; there is no undo pass
  - In READ COMMITTED, every statement (not every transaction) gets a fresh snapshot
- **Lab 001** now creates its own `mydb` and `orders`, uses `SHOW shared_memory_size` instead of `ipcs`, and disables autovacuum on its stale-statistics table
- **qa.md**: 14 of the original 28 answers corrected or extended; 11 questions added (39 total)
- **research/** folder created, with the experiment layout from 6.1
- **Next**: Lab 001 hands-on (unchanged)

---

## Topic Status Summary

| Phase | Topic | Sessions | Status |
|---|---|---|---|
| 1.1 | Architecture Overview | 1 | 🟡 In Progress |
| 1.2 | Process Model | — | ⬜ Not Started |
| 1.3 | Memory Architecture | — | ⬜ Not Started |
| 1.4 | Storage Layout | — | ⬜ Not Started |
| 1.5 | WAL | — | ⬜ Not Started |
| 1.6 | MVCC | — | ⬜ Not Started |
| 1.7 | VACUUM | — | ⬜ Not Started |
| 1.8 | Query Lifecycle | — | ⬜ Not Started |
| 1.9 | Index Internals | — | ⬜ Not Started |
| 1.10 | Lock Manager | — | ⬜ Not Started |

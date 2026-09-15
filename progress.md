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

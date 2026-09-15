# PostgreSQL Internals — Research & Learning Journal

> **Mission**: Go from "I can run queries" to "I can design, operate, and research HA PostgreSQL clusters at production scale."

## What This Repo Is

A **public**, **structured**, **honest** learning journal. Not polished docs — real notes, real mistakes, real lab outputs, real benchmarks. If you're a DBA, SRE, or backend engineer wanting to go deep on PostgreSQL internals, follow along.

## Lab Environment

- **PostgreSQL 16** (on-prem, physical machines)
- **OS**: Linux (bare metal)
- **Goal**: Build a 2-site synchronous HA cluster + conduct performance research on config parameters

## How This Is Organized

| Path | Contents |
|------|----------|
| `AGENT.md` | Agent context — rules, curriculum, session protocol |
| `progress.md` | Session log — what was covered, what's next |
| `qa.md` | Interview Q&A bank (DBA-level, public) |
| `backlog.md` | Parking lot — ideas parked for later |
| `curriculum/` | Study notes by topic and phase |
| `labs/` | Hands-on lab exercises with actual results |
| `research/` | Benchmarks, experiments, and paper draft |

## Curriculum Phases

1. **Internals** — Architecture, processes, memory, storage, WAL, MVCC, VACUUM, query planning, indexes, locks
2. **Configuration** — postgresql.conf, pg_hba.conf, SSL, PgBouncer, OS tuning
3. **Backup & Recovery** — pg_dump, pg_basebackup, PITR, pgBackRest
4. **Replication** — Streaming, sync/async, slots, logical replication
5. **HA & Failover** — Patroni, etcd, HAProxy, cross-DC design, failover drills
6. **Research** — Benchmarking, parameter experiments, publication

## Contributing / Following

This is a personal research repo made public for the community.
PRs for corrections are welcome. The `qa.md` file is intended to be a growing, free DBA interview question bank.

---

*Started: 2026-09-15 | PostgreSQL 16 | On-Prem Physical Cluster*

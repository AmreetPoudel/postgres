# Research

Raw data, experiment logs and the paper draft for Phase 6. Rule 13 in `AGENT.md`: every lab result and benchmark lands here, with the date and the PostgreSQL version.

| Path | Contents |
|---|---|
| `benchmarks/` | Benchmark scripts and raw output (pgbench logs, CSV, JSON) |
| `experiments/` | One folder per experiment: hypothesis, setup, raw results, analysis |
| `paper/` | The paper draft and its figures |

## Experiment folder template

Name folders `YYYYMMDD_short-name`, for example `20261101_shared_buffers/` (see `curriculum/6-research/6.1-benchmark-methodology.md`):

```
experiments/20261101_shared_buffers/
├── hypothesis.md      ← what you expected, written before running anything
├── setup.md           ← hardware, OS settings, PostgreSQL version, full config diff
├── results_raw.txt    ← untouched pgbench output
├── pg_stats.txt       ← pg_stat_database, pg_stat_bgwriter, pg_stat_wal snapshots
├── os_metrics.txt     ← iostat / vmstat during the run
└── analysis.md        ← what you found, and what it doesn't prove
```

Nothing here yet: the first entries arrive with Phase 6. Only real measurements go in this folder, never illustrative numbers.

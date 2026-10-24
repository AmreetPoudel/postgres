# Troubleshooting Labs — Index

Real production scenarios. Each lab has: the symptom, why it happens, how to diagnose, how to fix, and a hands-on exercise to reproduce and resolve it.

Do these AFTER completing the main curriculum. They assume you understand the internals.

**Paths and commands** assume a RHEL-style or source install, where everything lives in `$PGDATA`. On Debian/Ubuntu:
- Config files are in `/etc/postgresql/16/main/` (`SHOW config_file;` and `SHOW hba_file;` tell you)
- Logs are in `/var/log/postgresql/postgresql-16-main.log` (`SHOW log_directory;`)
- `pg_ctl` isn't on your PATH: use `pg_ctlcluster 16 main reload|restart` or `systemctl`
- Run the commands as the `postgres` OS user unless a step says otherwise

---

## By Severity

| Lab | Title | Severity | Freq |
|---|---|---|---|
| [T-01](t01-connection-exhaustion.md) | Connection exhaustion | P1 | Very common |
| [T-02](t02-long-running-tx-blocking.md) | Long transaction blocking everything | P1 | Very common |
| [T-03](t03-disk-full-pg-wal.md) | Disk full on pg_wal | P0 | Moderate |
| [T-04](t04-autovacuum-not-keeping-up.md) | Autovacuum not keeping up | P2→P1 | Common |
| [T-05](t05-stale-statistics-bad-plan.md) | Stale stats → bad query plan | P2 | Very common |
| [T-06](t06-replication-lag-spike.md) | Replication lag spike | P2 | Common |
| [T-07](t07-checkpoint-storm.md) | Checkpoint storm (I/O spikes) | P2 | Common |
| [T-08](t08-sequence-overflow.md) | Sequence overflow (int exhausted) | P0 | Moderate |
| [T-09](t09-temp-file-explosion.md) | Temp file explosion → disk full | P1 | Moderate |
| [T-10](t10-deadlock.md) | Deadlock storm | P2 | Moderate |
| [T-11](t11-xid-wraparound-emergency.md) | XID wraparound emergency | P0 | Rare / devastating |
| [T-12](t12-index-bloat.md) | Index bloat | P2 | Common |
| [T-13](t13-wal-archiving-failure.md) | WAL archiving failure (silent PITR break) | P1 | Moderate |
| [T-14](t14-pgbouncer-queue-buildup.md) | PgBouncer queue buildup | P1 | Common |
| [T-15](t15-missing-index.md) | Missing index on large table | P2 | Very common |
| [T-16](t16-pg-hba-lockout.md) | pg_hba.conf lockout | P0 | Moderate |
| [T-17](t17-query-plan-regression.md) | Query plan regression after upgrade | P2 | Moderate |
| [T-18](t18-oom-killed.md) | OOM killer killed PostgreSQL | P0 | Moderate |
| [T-19](t19-ssl-cert-expiry.md) | SSL certificate expiry | P0 | Moderate |
| [T-20](t20-table-corruption-detection.md) | Table/index corruption detection | P0 | Rare |
| [T-21](t21-hot-standby-conflict.md) | Hot standby query conflicts | P2 | Common |
| [T-22](t22-patroni-failover-debug.md) | Patroni failover didn't happen | P0 | Occasional |
| [T-23](t23-pitr-restore.md) | PITR after accidental data deletion | P1 | Occasional |
| [T-24](t24-logical-replication-stuck.md) | Logical replication subscription stuck | P2 | Moderate |
| [T-25](t25-slow-query-investigation.md) | Full slow query investigation workflow | Daily | Daily |

---

## By Category

### Connections & Locking
- T-01: Connection exhaustion
- T-02: Long transaction blocking (lock queue pile-up)
- T-10: Deadlock

### Disk & Storage
- T-03: Disk full on pg_wal (dead replication slot)
- T-09: Temp file explosion

### VACUUM & Bloat
- T-04: Autovacuum not keeping up
- T-11: XID wraparound emergency
- T-12: Index bloat

### Query Performance
- T-05: Stale statistics
- T-07: Checkpoint storm (write latency)
- T-15: Missing index
- T-17: Query plan regression
- T-25: Full slow query investigation

### Replication & HA
- T-06: Replication lag spike
- T-21: Hot standby query conflicts
- T-22: Patroni failover stuck
- T-24: Logical replication stuck

### Backup & Recovery
- T-13: WAL archiving failure
- T-23: PITR restore

### Configuration & Operations
- T-08: Sequence overflow
- T-14: PgBouncer queue buildup
- T-16: pg_hba.conf lockout
- T-18: OOM killer
- T-19: SSL cert expiry
- T-20: Corruption detection

---

## Recommended Order

If you're new to production ops, do them in this order:

1. T-25 (understand the investigation workflow first)
2. T-01, T-02, T-10 (connections/locking — most frequent)
3. T-05, T-15 (query performance — daily work)
4. T-04, T-12, T-11 (VACUUM chain — builds on each other)
5. T-07, T-09, T-03 (disk/I/O problems)
6. T-06, T-21, T-22, T-24 (replication — needs a cluster from Lab 004/005)
7. T-13, T-23 (backup/recovery — needs archiving configured)
8. T-08, T-14, T-16, T-17, T-18, T-19, T-20 (operational gotchas)

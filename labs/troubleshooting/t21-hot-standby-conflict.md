# T-21 — Hot Standby Query Conflicts

**Frequency**: Common on read-heavy standbys
**Severity**: P2 — queries killed on standby
**Related**: 4.1 Streaming Replication, 4.5 Replication Lag

---

## The Symptom

Queries on the standby get killed with:
```
ERROR: canceling statement due to conflict with recovery
DETAIL: User query might have needed to see row versions that must be removed.
```

---

## Why This Happens

When the primary runs VACUUM and removes dead tuples, it writes WAL records. The standby applies those WAL records. But if a standby query is still reading the rows that WAL wants to remove, there's a conflict.

PostgreSQL resolves it by canceling the standby query (after `max_standby_streaming_delay`).

---

## Diagnosis

```sql
-- On standby: check for conflict stats
SELECT confl_tablespace, confl_lock, confl_snapshot, confl_bufferpin, confl_deadlock
FROM pg_stat_database_conflicts WHERE datname = current_database();

-- confl_snapshot > 0 → queries being killed due to VACUUM conflicts
```

```bash
# Logs on standby:
grep "conflict with recovery" "$PGDATA"/log/postgresql-*.log | tail -10   # Debian/Ubuntu: /var/log/postgresql/
```

---

## Fix Options

**Option 1**: Increase delay before killing standby queries:
```
max_standby_streaming_delay = 120s    # give queries 2 minutes before killing them
```

**Option 2**: Enable hot_standby_feedback (tells primary what the standby needs):
```
# postgresql.conf on STANDBY
hot_standby_feedback = on
# Primary's autovacuum won't clean rows the standby still needs
# WARNING: can cause bloat on primary if standby has very long queries
```

**Option 3**: A dedicated reporting standby that never cancels queries. Replay waits for them instead, so this standby falls behind while long reports run. (A *delayed* standby doesn't help: conflicts happen whenever the WAL gets applied, delayed or not.)
```
# postgresql.conf on the reporting STANDBY
max_standby_streaming_delay = -1     # wait as long as it takes
```

---

## Lab Exercise (requires standby from Lab 004)

```sql
-- On PRIMARY: note autovacuum settings
SHOW autovacuum;
SHOW autovacuum_vacuum_scale_factor;
```

```sql
-- On STANDBY: check conflict counters before
SELECT confl_snapshot FROM pg_stat_database_conflicts
WHERE datname = current_database();

-- On STANDBY: run a long query
SELECT pg_sleep(120);
-- (This holds a snapshot open for 2 minutes, longer than the default
--  max_standby_streaming_delay of 30s)
```

```sql
-- On PRIMARY: while standby is sleeping, trigger a VACUUM that conflicts
CREATE TABLE conflict_test AS SELECT i, md5(i::text) FROM generate_series(1,100000) i;
UPDATE conflict_test SET md5 = md5(i::text);  -- create dead tuples
VACUUM conflict_test;   -- removes rows the standby's snapshot may still need
DROP TABLE conflict_test;
```

```sql
-- On STANDBY: check if confl_snapshot increased
SELECT confl_snapshot FROM pg_stat_database_conflicts
WHERE datname = current_database();
```

```
# Fix: add to postgresql.conf on STANDBY
hot_standby_feedback = on
```

**Write down**: Did confl_snapshot increase? ___  
What is the trade-off of hot_standby_feedback = on? ___

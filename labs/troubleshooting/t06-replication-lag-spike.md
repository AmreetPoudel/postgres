# T-06 — Replication Lag Spike

**Frequency in production**: Common
**Severity**: P2 — data loss risk if primary fails while lagging
**Related**: 4.1 Streaming Replication, 4.5 Replication Lag

---

## The Symptom

Alert: "Replication lag > 60 seconds" or "Standby WAL behind by 2GB".

---

## Why This Happens

1. Heavy write burst — WAL generated faster than standby can apply
2. Long query on standby blocking WAL apply (lock conflict)
3. Standby disk I/O saturated — applying WAL competes with read queries
4. Network bandwidth exhausted (cross-DC)

---

## Diagnosis

```sql
-- On PRIMARY: full lag breakdown
SELECT application_name, state,
       pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn))       AS total_lag,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(),sent_lsn)) AS unsent,
       write_lag, flush_lag, replay_lag
FROM pg_stat_replication;

-- On STANDBY: how old is last applied transaction?
SELECT now() - pg_last_xact_replay_timestamp() AS lag_time;

-- On STANDBY: is WAL apply paused?
SELECT pg_is_wal_replay_paused();

-- On STANDBY: queries blocking WAL apply?
SELECT pid, now() - query_start AS duration, wait_event_type, wait_event
FROM pg_stat_activity
WHERE wait_event IS NOT NULL
ORDER BY duration DESC LIMIT 10;
```

---

## Gap Analysis

```
sent vs current    → network bottleneck or wal_sender issue
write vs sent      → standby disk write speed
flush vs write     → standby fsync latency
replay vs flush    → standby CPU or query conflict with WAL apply
```

---

## Fix

```sql
-- On STANDBY: cancel a conflicting query if needed
SELECT pg_cancel_backend(pid)
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';
```

```
# postgresql.conf on STANDBY
max_standby_streaming_delay = 30s  -- auto-cancel conflicting queries
```

```
# postgresql.conf on PRIMARY
wal_compression = on               -- reduce WAL size sent over network
```

---

## Lab Exercise (requires a standby from Lab 004)

```sql
-- On STANDBY: pause WAL replay to simulate lag
SELECT pg_wal_replay_pause();
SELECT pg_is_wal_replay_paused();   -- true
```

```sql
-- On PRIMARY: generate WAL while replay is paused
CREATE TABLE lag_demo AS
SELECT md5(i::text) FROM generate_series(1, 500000) i;

-- Check lag on PRIMARY
SELECT application_name,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) AS lag
FROM pg_stat_replication;
```

```sql
-- On STANDBY: resume
SELECT pg_wal_replay_resume();

-- Watch lag drain on PRIMARY
SELECT application_name, replay_lag FROM pg_stat_replication;
```

**Write down**: Peak lag size: ___  
Time to catch up after resume: ___

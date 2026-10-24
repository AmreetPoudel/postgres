# T-03 — Disk Full on pg_wal

**Frequency in production**: Moderate — catastrophic when it hits
**Severity**: P0 — database stops accepting all writes
**Related**: 1.5 WAL, 4.3 Replication Slots, 2.6 Checkpoint Config

---

## The Symptom

```
PANIC: could not write to file "pg_wal/...": No space left on device
```

All writes stop. Reads may still work from shared_buffers cache.

---

## Why This Happens

1. **Dead replication slot holding WAL** — standby offline, slot keeps accumulating WAL forever
2. **Archiving failure** — `archive_command` fails, WAL piles up waiting to archive
3. **Long-running transaction** — prevents WAL recycling
4. **max_wal_size too small** — burst of writes fills WAL faster than checkpoints clear it

---

## Diagnosis

```bash
# 1. Check disk usage
df -h
du -sh "$PGDATA/pg_wal/"
ls "$PGDATA/pg_wal/" | wc -l   # normal: 20-50. alarming: thousands
```

```sql
-- 2. Check replication slots (most common culprit)
SELECT slot_name, active,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS wal_retained
FROM pg_replication_slots
ORDER BY pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) DESC;

-- 3. Check archiving
SELECT last_archived_wal, now() - last_archived_time AS since_archived,
       failed_count, last_failed_wal
FROM pg_stat_archiver;

-- 4. Oldest transaction blocking WAL recycling
SELECT pid, now() - xact_start AS age, left(query, 80)
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY age DESC LIMIT 5;
```

---

## Immediate Fix

```sql
-- Drop dead replication slot (standby must be rebuilt after)
SELECT pg_drop_replication_slot('dead_slot_name');
```

```bash
# Fix archive_command and reload
pg_ctl reload -D "$PGDATA"
```

---

## Prevention

```
# postgresql.conf (PG13+)
max_slot_wal_keep_size = 10GB   -- hard limit: invalidate slot before disk fills
```

```sql
-- Alert query (run daily)
SELECT slot_name, active,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained
FROM pg_replication_slots;
-- Alert if retained > 5GB
```

---

## Lab Exercise

```sql
-- Create a slot that simulates a dead standby
SELECT pg_create_physical_replication_slot('zombie_standby');

-- Check initial WAL retained (near 0)
SELECT slot_name,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained
FROM pg_replication_slots WHERE slot_name = 'zombie_standby';

-- Generate WAL (the slot will now retain all of it)
CREATE TABLE wal_gen AS SELECT md5(i::text) d FROM generate_series(1,200000) i;
DROP TABLE wal_gen;

-- Check how much WAL the slot is now holding
SELECT slot_name,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained
FROM pg_replication_slots WHERE slot_name = 'zombie_standby';

-- Clean up
SELECT pg_drop_replication_slot('zombie_standby');
```

**Write down**: How much WAL was retained after generating data? ___  
What would happen in production if you left this slot for 1 week? ___

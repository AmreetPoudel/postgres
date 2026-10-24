# T-03 — Disk Full on pg_wal

**Frequency in production**: Moderate — catastrophic when it hits
**Severity**: P0 — database stops accepting all writes
**Related**: 1.5 WAL, 4.3 Replication Slots, 2.6 Checkpoint Config

---

## The Symptom

```
PANIC: could not write to file "pg_wal/...": No space left on device
```

PostgreSQL crashes, and it can't start again until there's free space, because starting up has to write WAL too.

---

## Why This Happens

1. **Dead replication slot holding WAL** — standby offline, slot keeps accumulating WAL forever
2. **Archiving failure** — `archive_command` fails, WAL piles up waiting to archive
3. **`wal_keep_size` too large** for the disk — PostgreSQL keeps that much extra WAL on purpose
4. **`max_wal_size` bigger than the disk can hold** — it's a soft limit, and a heavy write burst or a slow checkpoint can push WAL beyond it

(A long-running transaction doesn't hold WAL back. It holds back VACUUM, which is a different problem: T-04.)

---

## Diagnosis

```bash
# 1. Check disk usage
df -h
du -sh "$PGDATA/pg_wal/"
ls "$PGDATA/pg_wal/" | wc -l   # normal: 20-50. alarming: thousands
ls "$PGDATA/pg_wal/archive_status/"*.ready 2>/dev/null | wc -l   # segments still waiting to be archived
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

-- 4. Settings that keep WAL around on purpose, and checkpoint pressure
--    (old WAL is only removed after a checkpoint)
SHOW wal_keep_size;
SHOW max_wal_size;
SHOW max_slot_wal_keep_size;
SELECT checkpoints_timed, checkpoints_req FROM pg_stat_bgwriter;
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

**Never delete files in `pg_wal/` by hand** to free space: that corrupts the cluster. Free space elsewhere on the volume, drop the dead slot, or fix archiving, and let the next checkpoint remove the WAL.

---

## Prevention

```
# postgresql.conf (PG13+)
max_slot_wal_keep_size = 10GB   # hard limit: invalidate the slot before the disk fills
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
-- Create a slot that simulates a dead standby. The second argument (true) makes it
-- reserve WAL right away; without it the slot holds nothing until a standby connects.
SELECT pg_create_physical_replication_slot('zombie_standby', true);

-- Check initial WAL retained (it counts from the last checkpoint, so not exactly 0)
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

-- The slot's own view of the danger (PG13+)
SELECT slot_name, wal_status, pg_size_pretty(safe_wal_size) AS safe_wal_left
FROM pg_replication_slots WHERE slot_name = 'zombie_standby';

-- Clean up
SELECT pg_drop_replication_slot('zombie_standby');
```

**Write down**: How much WAL was retained after generating data? ___  
What would happen in production if you left this slot for 1 week? ___

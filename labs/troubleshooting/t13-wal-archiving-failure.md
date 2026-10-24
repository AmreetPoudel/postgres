# T-13 — WAL Archiving Failure: PITR Broken Silently

**Frequency**: Moderate
**Severity**: P1 — you think you have PITR but you don't
**Related**: 1.5 WAL, 3.3 PITR, 3.4 pgBackRest

---

## The Symptom

Alerts: "WAL archive failures > 10". Or worse — no alert, and you only discover during a recovery attempt that your archive has gaps.

---

## Why This Happens

`archive_command` runs as a shell command. It fails, logging only a WARNING that nobody reads, if:
- Archive destination is full
- NFS mount dropped
- SSH key expired
- Wrong permissions on archive directory
- The command itself has a bug

PostgreSQL keeps the WAL segment and retries: three quick attempts, then again about a minute later, forever, incrementing `failed_count` in `pg_stat_archiver` each time. (`archive_timeout` has nothing to do with retries.) It never stops the database — until `pg_wal/` fills the disk (T-03).

---

## Diagnosis

```sql
SELECT archived_count,
       failed_count,
       last_archived_wal,
       now() - last_archived_time  AS since_last_archive,
       last_failed_wal,
       now() - last_failed_time    AS since_last_failure
FROM pg_stat_archiver;

-- Red flags:
-- last_failed_time newer than last_archived_time → the latest attempt failed
-- since_last_archive > 10 minutes on a busy server
-- (failed_count only ever grows, so on its own it can't tell you "broken right now")
```

```bash
# Test your archive_command manually right now
archive_cmd=$(psql -t -c "SHOW archive_command;")
echo "Command: $archive_cmd"

# Segments waiting to be archived have a .ready file; the oldest is next in line
ls "$PGDATA/pg_wal/archive_status/"*.ready 2>/dev/null | head -5
ls "$PGDATA/pg_wal/archive_status/"*.ready 2>/dev/null | wc -l
```

---

## Fix

```bash
# 1. Fix the underlying issue (disk space, mount, permissions, the command itself)
# 2. Do NOT copy segments into the archive by hand. PostgreSQL retries on its own
#    within about a minute, and a hand-copied file makes a "test ! -f" style
#    archive_command refuse that segment forever. Just watch it catch up:
psql -c "SELECT last_archived_wal, last_archived_time, last_failed_time FROM pg_stat_archiver;"
ls "$PGDATA/pg_wal/archive_status/"*.ready 2>/dev/null | wc -l    # should drop to 0

# 3. Optionally reset the counters so the next failure stands out:
psql -c "SELECT pg_stat_reset_shared('archiver');"
```

---

## Prevention

```
# postgresql.conf
archive_timeout = 300s          # switch segments after 5 min if any WAL was written,
                                # bounding how much recent data a PITR could lose
```

Always verify archiving works after setup:
```bash
# Force a WAL segment switch (it does nothing if no WAL was written since the last one)
psql -c "SELECT pg_switch_wal();"
sleep 10
psql -c "SELECT * FROM pg_stat_archiver;"
# last_archived_wal should now be the segment you just switched away from
```

---

## Lab Exercise

```sql
-- Check your current archiver status
SELECT archived_count, failed_count,
       last_archived_wal, now() - last_archived_time AS since_archived,
       last_failed_wal
FROM pg_stat_archiver;
```

```bash
# Simulate archive failure: set an archive_command that always fails
psql -c "ALTER SYSTEM SET archive_mode = on;"
psql -c "ALTER SYSTEM SET archive_command = 'false';"  # 'false' always returns exit 1
pg_ctl restart -D "$PGDATA" -m fast

# Generate a WAL switch
psql -c "SELECT pg_switch_wal();"
sleep 15

# Check failure count
psql -c "SELECT archived_count, failed_count, last_failed_wal FROM pg_stat_archiver;"
```

```bash
# Fix: set a working archive_command
mkdir -p /tmp/pg_archive
psql -c "ALTER SYSTEM SET archive_command = 'cp %p /tmp/pg_archive/%f';"
pg_ctl reload -D "$PGDATA"
psql -c "SELECT pg_switch_wal();"
sleep 15
psql -c "SELECT archived_count, failed_count FROM pg_stat_archiver;"
ls /tmp/pg_archive/ | head -5

# Clean up
psql -c "ALTER SYSTEM RESET archive_mode;"
psql -c "ALTER SYSTEM RESET archive_command;"
pg_ctl restart -D "$PGDATA" -m fast
```

**Write down**: How long before failure was detected? ___  
How would you alert on this in Prometheus? ___

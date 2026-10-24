# T-13 — WAL Archiving Failure: PITR Broken Silently

**Frequency**: Moderate
**Severity**: P1 — you think you have PITR but you don't
**Related**: 1.5 WAL, 3.3 PITR, 3.4 pgBackRest

---

## The Symptom

Alerts: "WAL archive failures > 10". Or worse — no alert, and you only discover during a recovery attempt that your archive has gaps.

---

## Why This Happens

`archive_command` runs as a shell command. It fails silently if:
- Archive destination is full
- NFS mount dropped
- SSH key expired
- Wrong permissions on archive directory
- The command itself has a bug

PostgreSQL retries the archive command every `archive_timeout` seconds, keeps the WAL segment, and increments `failed_count` in `pg_stat_archiver`. But it doesn't stop the database.

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
-- failed_count > 0
-- since_last_archive > 10 minutes
-- last_failed_wal is not null
```

```bash
# Test your archive_command manually right now
archive_cmd=$(psql -t -c "SHOW archive_command;")
echo "Command: $archive_cmd"

# Find the oldest un-archived WAL
ls -la "$PGDATA/pg_wal/" | head -20
```

---

## Fix

```bash
# 1. Fix the underlying issue (disk space, mount, permissions)
# 2. Manually archive the failed file:
cp "$PGDATA/pg_wal/FAILED_WAL_SEGMENT" /archive/location/

# 3. Reset the archiver stats:
SELECT pg_stat_reset_shared('archiver');
```

---

## Prevention

```
# postgresql.conf
archive_timeout = 300s          -- force WAL segment every 5 min even if idle
                                 -- limits data loss and archive gaps
```

Always verify archiving works after setup:
```bash
# Create a checkpoint to force WAL switch
psql -c "SELECT pg_switch_wal();"
sleep 10
psql -c "SELECT * FROM pg_stat_archiver;"
# failed_count must be 0
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

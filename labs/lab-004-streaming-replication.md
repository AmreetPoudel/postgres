# Lab 004 — Streaming Replication Setup

**Topics**: 4.1 Streaming Replication, 4.3 Replication Slots
**Read first**: `curriculum/4-replication/4.1-streaming-replication.md`, `curriculum/4-replication/4.3-replication-slots.md`
**Time**: ~90 minutes
**Requires**: Two servers (or VMs): PRIMARY_IP and STANDBY_IP

---

## Goal

By the end of this lab you will have:
- A primary accepting writes
- A standby streaming WAL from primary
- Verified data written on primary appears on standby
- Measured replication lag
- Tested manual failover (promotion)
- Rebuilt the old primary as a standby

---

## Architecture

```
PRIMARY  (e.g., 10.0.0.10)  —— WAL stream ——►  STANDBY (e.g., 10.0.0.20)
```

Replace IP addresses below with your actual server IPs.

---

## Task 1: Configure the Primary

```bash
# On PRIMARY server — as root or postgres user
```

```sql
-- Edit postgresql.conf
ALTER SYSTEM SET wal_level           = 'replica';
ALTER SYSTEM SET max_wal_senders     = 5;
ALTER SYSTEM SET wal_keep_size       = '1GB';
ALTER SYSTEM SET hot_standby         = on;
ALTER SYSTEM SET max_replication_slots = 5;
```

```bash
# Restart to apply wal_level (requires restart)
pg_ctl restart -D "$PGDATA" -m fast
```

```sql
-- Create replication user
CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'ReplPass123!';

-- Verify
SELECT usename, replication FROM pg_user WHERE usename = 'replicator';
```

```bash
# Add standby to pg_hba.conf (replace STANDBY_IP)
echo "host replication replicator STANDBY_IP/32 scram-sha-256" >> "$PGDATA/pg_hba.conf"
pg_ctl reload -D "$PGDATA"
```

---

## Task 2: Create Base Backup on Standby

```bash
# On STANDBY server
# Stop any existing PostgreSQL
pg_ctl stop -D "$PGDATA" -m fast 2>/dev/null || true

# Clear data directory (pg_basebackup requires empty target)
rm -rf "$PGDATA"/*

# Take base backup from primary
# -R flag: writes standby.signal + primary_conninfo automatically
pg_basebackup \
    -h PRIMARY_IP \
    -U replicator \
    -W \
    -D "$PGDATA" \
    -R \
    -X stream \
    --checkpoint=fast \
    -P -v

# Verify standby.signal was created (makes PostgreSQL start in standby mode)
ls -la "$PGDATA/standby.signal"

# Verify primary_conninfo was written
grep primary_conninfo "$PGDATA/postgresql.auto.conf"
```

---

## Task 3: Start the Standby

```bash
# On STANDBY server
pg_ctl start -D "$PGDATA"

# Watch logs for "started streaming"
tail -20 "$PGDATA"/log/postgresql-*.log | grep -i -E "stream|recovery|standby"
```

```sql
-- Verify standby is in recovery mode
SELECT pg_is_in_recovery();          -- must return: true

-- Check lag from primary
SELECT now() - pg_last_xact_replay_timestamp() AS lag_from_primary;
```

---

## Task 4: Verify Replication from Both Sides

### On PRIMARY:

```sql
-- See the standby in pg_stat_replication
SELECT
    application_name,
    client_addr,
    state,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes,
    replay_lag,
    sync_state
FROM pg_stat_replication;
```

Record: state = _____ | lag_bytes = _____ | sync_state = _____

### Test data flows:

```sql
-- On PRIMARY: write some data
CREATE TABLE repl_test (id serial PRIMARY KEY, msg text, ts timestamp DEFAULT now());
INSERT INTO repl_test (msg) VALUES ('row one'), ('row two'), ('row three');
```

```sql
-- On STANDBY: immediately read it
SELECT * FROM repl_test;
-- All 3 rows should appear
```

```sql
-- On STANDBY: prove it is read-only
INSERT INTO repl_test (msg) VALUES ('standby write');
-- Must get: ERROR: cannot execute INSERT in a read-only transaction
```

---

## Task 5: Create a Replication Slot

```sql
-- On PRIMARY: create a named physical slot for the standby
SELECT pg_create_physical_replication_slot('standby_slot');

-- Verify it was created
SELECT slot_name, slot_type, active, restart_lsn FROM pg_replication_slots;
```

```bash
# On STANDBY: tell it to use the slot
echo "primary_slot_name = 'standby_slot'" >> "$PGDATA/postgresql.auto.conf"
pg_ctl restart -D "$PGDATA" -m fast
```

```sql
-- On PRIMARY: verify slot is now active and being used
SELECT slot_name, active,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS wal_retained
FROM pg_replication_slots;
-- active = true, wal_retained should be small
```

---

## Task 6: Manual Failover Drill

**DRILL: simulate primary failure and promote standby**

```bash
# Step 1: Stop primary (simulate crash)
# On PRIMARY:
pg_ctl stop -D "$PGDATA" -m immediate   # kills it like kill -9

# Step 2: Promote standby to primary
# On STANDBY:
pg_ctl promote -D "$PGDATA"

# Step 3: Verify promotion
# On STANDBY (now primary):
psql -c "SELECT pg_is_in_recovery();"   # must return: false
psql -c "INSERT INTO repl_test (msg) VALUES ('after promotion');"   # must succeed
psql -c "SELECT * FROM repl_test;"
```

Record: How many seconds did promotion take? _____

### Rebuild old primary as standby:

```bash
# On OLD PRIMARY server:
pg_ctl stop -D "$PGDATA" -m fast 2>/dev/null || true
rm -rf "$PGDATA"/*

pg_basebackup \
    -h STANDBY_IP \      # old standby is now new primary
    -U replicator \
    -W \
    -D "$PGDATA" \
    -R -X stream --checkpoint=fast -P -v

pg_ctl start -D "$PGDATA"
```

```sql
-- On NEW PRIMARY (old standby): verify both nodes are connected
SELECT application_name, state, sync_state FROM pg_stat_replication;
```

---

## Completion Checklist

- [ ] Primary configured: wal_level=replica, replicator user, pg_hba.conf entry
- [ ] Standby created via pg_basebackup -R
- [ ] Standby started and streaming (pg_is_in_recovery() = true)
- [ ] Data written on primary visible on standby
- [ ] Confirmed standby is read-only (INSERT fails)
- [ ] Created replication slot and configured standby to use it
- [ ] Ran failover drill: stopped primary, promoted standby
- [ ] Rebuilt old primary as standby of new primary
- [ ] Updated `progress.md`

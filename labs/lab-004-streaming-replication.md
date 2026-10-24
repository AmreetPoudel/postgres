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
-- ALTER SYSTEM writes these into postgresql.auto.conf
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
SELECT rolname, rolreplication FROM pg_roles WHERE rolname = 'replicator';
```

```bash
# Add standby to pg_hba.conf (replace STANDBY_IP). Ask PostgreSQL where the file is:
# on Debian/Ubuntu it's /etc/postgresql/16/main/pg_hba.conf, not in $PGDATA.
# If your pg_hba.conf ends with a reject-all rule, put this line ABOVE it instead.
HBA=$(psql -U postgres -tAc "SHOW hba_file;")
echo "host replication replicator STANDBY_IP/32 scram-sha-256" >> "$HBA"
psql -U postgres -c "SELECT pg_reload_conf();"
```

---

## Task 2: Create Base Backup on Standby

```bash
# On STANDBY server
# Stop any existing PostgreSQL
pg_ctl stop -D "$PGDATA" -m fast 2>/dev/null || true

# Clear data directory (pg_basebackup requires an empty target).
# ${PGDATA:?} aborts if PGDATA is unset, so this can never turn into rm -rf /*
rm -rf "${PGDATA:?}"/*

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

# Watch logs for "started streaming" (Debian/Ubuntu: /var/log/postgresql/postgresql-16-main.log)
tail -20 "$PGDATA"/log/postgresql-*.log | grep -i -E "stream|recovery|standby"
```

```sql
-- Verify standby is in recovery mode
SELECT pg_is_in_recovery();          -- must return: true

-- Time since the last replayed transaction (it grows on an idle primary even with zero lag)
SELECT now() - pg_last_xact_replay_timestamp() AS since_last_replay;
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
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes,
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
SELECT pg_create_physical_replication_slot('standby_slot', true);   -- true = reserve WAL now

-- Verify it was created
SELECT slot_name, slot_type, active, restart_lsn FROM pg_replication_slots;
```

```bash
# On STANDBY: tell it to use the slot. ALTER SYSTEM works on a standby too
# (don't edit postgresql.auto.conf by hand), and since PG13 a reload is enough.
psql -U postgres -c "ALTER SYSTEM SET primary_slot_name = 'standby_slot';"
psql -U postgres -c "SELECT pg_reload_conf();"
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
pg_ctl stop -D "$PGDATA" -m immediate   # no shutdown checkpoint: like a crash

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
# On the NEW PRIMARY (the old standby) first: let the old primary connect for replication.
# Its pg_hba.conf has a rule for STANDBY_IP (or no rule at all on Debian, where the file
# lives outside PGDATA), but none for PRIMARY_IP. Put it above any reject-all line.
HBA=$(psql -U postgres -tAc "SHOW hba_file;")
echo "host replication replicator PRIMARY_IP/32 scram-sha-256" >> "$HBA"
psql -U postgres -c "SELECT pg_reload_conf();"
```

```bash
# On OLD PRIMARY server (STANDBY_IP = the old standby, now the primary):
pg_ctl stop -D "$PGDATA" -m fast 2>/dev/null || true
rm -rf "${PGDATA:?}"/*

# -C -S creates a replication slot on the new primary; -R records it in the config
pg_basebackup \
    -h STANDBY_IP \
    -U replicator \
    -W \
    -D "$PGDATA" \
    -C -S old_primary_slot \
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

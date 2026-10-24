# T-16 — pg_hba.conf Misconfiguration Lockout

**Frequency**: Moderate (happens during infra changes)
**Severity**: P0 — nobody can connect
**Related**: 2.2 pg_hba.conf

---

## The Symptom

After editing pg_hba.conf and reloading, all application connections fail:
```
FATAL: pg_hba.conf rejects connection for host "10.0.0.5", user "appuser", database "mydb", no encryption
```
or, when no rule matches at all:
```
FATAL: no pg_hba.conf entry for host "10.0.0.5", user "appuser", database "mydb", no encryption
```

---

## Why This Happens

pg_hba.conf is processed top-to-bottom, first match wins. Common mistakes:
- Added new rule at the bottom below a `reject all` line
- Deleted the line that allowed your app's IP range
- Changed `md5` to `scram-sha-256` while some users still have md5-hashed passwords (or a very old client library can't do SCRAM)
- Typo in IP address

---

## Diagnosis

```bash
# Which file is in use? (if you can still connect at all)
psql -U postgres -tAc "SHOW hba_file;"
cat /etc/postgresql/16/main/pg_hba.conf    # Debian/Ubuntu
cat "$PGDATA/pg_hba.conf"                  # RHEL and source installs

# Can the superuser still connect over the Unix socket? That depends on your
# "local" lines: they're rules too (the lab below breaks them as well)
psql -U postgres -h /var/run/postgresql
```

```sql
-- The rules as the file reads RIGHT NOW: parsed fresh, not necessarily what the
-- server has loaded. That makes this view a way to test changes before a reload.
SELECT line_number, type, database, user_name, address, auth_method, error
FROM pg_hba_file_rules ORDER BY line_number;
```

---

## Fix

```bash
# Edit pg_hba.conf to fix the rule
vim "$PGDATA/pg_hba.conf"

# Test the new file BEFORE reloading: any row here means a broken line
psql -c "SELECT line_number, error FROM pg_hba_file_rules WHERE error IS NOT NULL;"

# Then reload. pg_ctl reload just sends a signal, so it works even when nobody
# can log in (Debian/Ubuntu: pg_ctlcluster 16 main reload)
pg_ctl reload -D "$PGDATA"
```

---

## Lab Exercise

```bash
# Find and back up the current pg_hba.conf (do this BEFORE breaking anything)
HBA=$(psql -U postgres -tAc "SHOW hba_file;")
cp "$HBA" /tmp/pg_hba.conf.backup
```

```sql
-- See current rules
SELECT line_number, type, database, user_name, address, auth_method
FROM pg_hba_file_rules ORDER BY line_number;
```

```bash
# Simulate a lockout: add a REJECT rule at the top
cat > /tmp/bad_hba.conf << 'EOF'
# DO NOT USE: demo only
local   all   all                reject
host    all   all   0.0.0.0/0   reject
EOF

cat /tmp/bad_hba.conf "$HBA" > /tmp/combined.conf
cp /tmp/combined.conf "$HBA"
pg_ctl reload -D "$PGDATA"      # Debian/Ubuntu: pg_ctlcluster 16 main reload
```

```bash
# Try to connect — should fail
psql -U postgres -h 127.0.0.1 -d postgres -c "SELECT 1;" 2>&1
# FATAL: pg_hba.conf rejects connection...

# Fix: restore from backup
cp /tmp/pg_hba.conf.backup "$HBA"
pg_ctl reload -D "$PGDATA"      # works without any database connection

# Verify recovery
psql -U postgres -h 127.0.0.1 -d postgres -c "SELECT 1;"
# Should return: 1
```

**Write down**: What error message did the failed connection show? ___  
How did you recover without a restart? ___

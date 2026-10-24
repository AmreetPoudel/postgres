# T-16 — pg_hba.conf Misconfiguration Lockout

**Frequency**: Moderate (happens during infra changes)
**Severity**: P0 — nobody can connect
**Related**: 2.2 pg_hba.conf

---

## The Symptom

After editing pg_hba.conf and reloading, all application connections fail:
```
FATAL: pg_hba.conf rejects connection for host "10.0.0.5", user "appuser", database "mydb"
```

---

## Why This Happens

pg_hba.conf is processed top-to-bottom, first match wins. Common mistakes:
- Added new rule at the bottom below a `reject all` line
- Deleted the line that allowed your app's IP range
- Changed `md5` to `scram-sha-256` but client doesn't support it
- Typo in IP address

---

## Diagnosis

```bash
# Check current pg_hba.conf
cat "$PGDATA/pg_hba.conf"

# Test: can superuser still connect locally?
psql -U postgres -h /var/run/postgresql   # local socket — unaffected by network rules
```

```sql
-- See the loaded hba rules (PG10+)
SELECT line_number, type, database, user_name, address, auth_method
FROM pg_hba_file_rules ORDER BY line_number;
```

---

## Fix

```bash
# Edit pg_hba.conf to fix the rule
vim "$PGDATA/pg_hba.conf"

# Test the new config BEFORE reloading
pg_ctl reload -D "$PGDATA"

# Verify in pg_hba_file_rules again:
psql -c "SELECT * FROM pg_hba_file_rules;"
```

---

## Lab Exercise

```bash
# Back up current pg_hba.conf
cp "$PGDATA/pg_hba.conf" /tmp/pg_hba.conf.backup
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

cat /tmp/bad_hba.conf "$PGDATA/pg_hba.conf" > /tmp/combined.conf
cp /tmp/combined.conf "$PGDATA/pg_hba.conf"
pg_ctl reload -D "$PGDATA"
```

```bash
# Try to connect — should fail
psql -U postgres -h 127.0.0.1 -d postgres -c "SELECT 1;" 2>&1
# FATAL: pg_hba.conf rejects connection...

# Fix: restore from backup
cp /tmp/pg_hba.conf.backup "$PGDATA/pg_hba.conf"
pg_ctl reload -D "$PGDATA"

# Verify recovery
psql -U postgres -h 127.0.0.1 -d postgres -c "SELECT 1;"
# Should return: 1
```

**Write down**: What error message did the failed connection show? ___  
How did you recover without a restart? ___

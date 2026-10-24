# T-19 — SSL Certificate Expiry: All Connections Fail

**Frequency**: Moderate — usually at 3 AM on a holiday
**Severity**: P0 — encrypted connections refused
**Related**: 2.3 SSL/TLS

---

## The Symptom

Applications using SSL to connect get:
```
SSL error: certificate has expired
```
or:
```
FATAL: could not accept SSL connection: certificate verify failed
```

---

## Why This Happens

`server.crt` (and/or the CA certificate) expired. PostgreSQL loads the certificate at startup and at every reload. Once it has expired, clients that verify it (`sslmode=verify-ca` or `verify-full`) refuse to connect. Clients using `sslmode=require` don't check the certificate at all, so they keep working, which is why expiry often surprises only some of your applications.

---

## Diagnosis

```bash
# Which file? psql -c "SHOW ssl_cert_file;"  A relative name like server.crt lives in
# $PGDATA; Debian/Ubuntu default to /etc/ssl/certs/ssl-cert-snakeoil.pem
# Check certificate expiry
openssl x509 -in "$PGDATA/server.crt" -noout -dates
# notAfter=Oct 14 00:00:00 2026 GMT  ← if this is in the past, you're expired

# How long until expiry?
openssl x509 -in "$PGDATA/server.crt" -noout -checkend 0
# exits 1 if already expired

openssl x509 -in "$PGDATA/server.crt" -noout -checkend 2592000
# exits 1 if expires within 30 days
```

```sql
-- What SSL parameters are configured?
SELECT name, setting FROM pg_settings WHERE name LIKE 'ssl%';
```

---

## Fix

```bash
# Regenerate self-signed certificate (development/internal use)
openssl req -new -x509 -days 365 -nodes \
    -keyout "$PGDATA/server.key" \
    -out "$PGDATA/server.crt" \
    -subj "/CN=postgres-server/O=myorg"

chmod 600 "$PGDATA/server.key"
chown postgres:postgres "$PGDATA/server.key" "$PGDATA/server.crt"

# Reload PostgreSQL (no restart needed: certificates are reloaded on SIGHUP since PG10)
pg_ctl reload -D "$PGDATA"
```

---

## Prevention

```bash
# Cron job: alert 30 days before expiry
0 6 * * * openssl x509 -in /var/lib/postgresql/16/main/server.crt \
    -noout -checkend 2592000 || \
    echo "PostgreSQL SSL cert expires within 30 days!" | mail -s "CERT ALERT" ops@myorg.com
```

---

## Lab Exercise

```bash
# Check your current certificate
openssl x509 -in "$PGDATA/server.crt" -noout -dates 2>/dev/null || \
    echo "No SSL cert found — SSL may be disabled"

# Check if SSL is enabled
psql -c "SHOW ssl;"

# If SSL is enabled, check days until expiry:
openssl x509 -in "$PGDATA/server.crt" -noout -enddate 2>/dev/null
```

```bash
# Generate a test certificate that expires in 1 day (to test near-expiry detection)
mkdir -p /tmp/ssl_test
openssl req -new -x509 -days 1 -nodes \
    -keyout /tmp/ssl_test/server.key \
    -out /tmp/ssl_test/server.crt \
    -subj "/CN=test" 2>/dev/null

# Check it with the 30-day check:
openssl x509 -in /tmp/ssl_test/server.crt -noout -checkend 2592000
echo "Exit code: $?"  # 1 = expires within 30 days = needs renewal

# How many days left?
openssl x509 -in /tmp/ssl_test/server.crt -noout -enddate
```

**Write down**: Does your PostgreSQL have SSL enabled? ___  
When does the cert expire? ___  
What exit code means "cert expires within 30 days"? ___

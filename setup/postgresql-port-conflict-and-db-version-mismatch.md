# PostgreSQL Port Conflict & Database Version Mismatch on Odoo Startup

| Field         | Value                                      |
|---------------|---------------------------------------------|
| Category      | setup                                       |
| Odoo Versions | 17, 18, 19                                  |
| Severity      | 🔴 Critical                                 |
| Last Verified | 2026-05-30                                  |
| Author        | ENG/Gamal Mansour                           |

**Tags:** `postgresql`, `port-conflict`, `startup`, `database`, `version-mismatch`, `short_time_format`, `brew`, `macos`

---

## Problem

Two separate issues often appear together when running multiple Odoo versions on the same macOS machine:

### Issue 1 — PostgreSQL Port Conflict (5432 already in use)

```
FATAL:  could not create any TCP/IP sockets
could not bind IPv4 address "127.0.0.1": Address already in use
HINT:  Is another postmaster already running on port 5432?
```

`brew services list` shows:
```
postgresql@18    error    1    gamal ~/Library/LaunchAgents/homebrew.mxcl.postgresql@18.plist
```

### Issue 2 — Column Missing Due to DB Version Mismatch

```
ERROR:  column res_lang.short_time_format does not exist
LINE 1: ..."res_lang"."time_format", "res_lang"."short_time_format"...

WARNING: Skipping database test_pure as its base version is not 18.0.1.3.
```

---

## Root Cause

### Issue 1
On macOS with Homebrew, multiple PostgreSQL versions (e.g., @13, @14, @18) can be installed.
If `postgresql@13` (or any older version) is already running on port `5432` and `postgresql@18` is set
to auto-start via `brew services`, it will fail with a port conflict every time. The older PostgreSQL
process holds port 5432 and refuses the newer one.

### Issue 2
A database created with **Odoo 19** (which includes `short_time_format` column in `res_lang`) was
accidentally opened against an **Odoo 18** instance. Odoo 18's code expects this column too (added
in a patch release), but the DB schema is either:
- From a **newer** Odoo version (downgrade attempt — not supported)
- From an **older** Odoo 18 build that predates the column addition

The `ir_cron` warning `base version is not 18.0.1.3` is a reliable indicator of a cross-version DB issue.

---

## Solution ✅

### Fix 1 — PostgreSQL Port Conflict

**Step 1:** Identify which PostgreSQL process owns port 5432:
```bash
lsof -i :5432
ps aux | grep postgres | grep -v grep
```

**Step 2:** Stop the conflicting brew service (the one causing the error):
```bash
brew services stop postgresql@18   # stop whichever version is erroring
```

**Step 3:** Verify the correct PostgreSQL is running:
```bash
brew services list | grep postgres
lsof -i :5432
```

> ⚠️ **DO NOT** stop the PostgreSQL version that is actually serving your databases.
> Identify which version owns your data using `ps aux | grep postgres`.

---

### Fix 2 — Database Version Mismatch (`short_time_format` missing)

**Step 1:** Identify the DB's Odoo version:
```bash
psql -U odoo -h localhost -d <db_name> -c "SELECT name, latest_version FROM ir_module_module WHERE name = 'base';"
```

**Step 2A — If the DB is expendable (test/dev):** Drop it:
```bash
# Terminate active connections first
psql -U odoo -h localhost -d postgres -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname = '<db_name>' AND pid <> pg_backend_pid();"

# Then drop
dropdb -U odoo -h localhost <db_name>
```

**Step 2B — If the DB must be preserved:** Run a module upgrade to let Odoo add the missing column:
```bash
.venv/bin/python odoo-bin -c odoo18_dev.conf -d <db_name> -u base --stop-after-init
```

**Step 2C — Emergency quick fix (use with caution):**
```sql
ALTER TABLE res_lang ADD COLUMN IF NOT EXISTS short_time_format VARCHAR DEFAULT 'HH:MM:SS';
UPDATE ir_module_module SET latest_version = '18.0.1.3' WHERE name = 'base';
```

---

## ⚠️ Pitfalls

- **Never downgrade a DB**: Opening an Odoo 19 DB with Odoo 18 is unsupported. The schema will have
  columns that Odoo 18's migrations never ran, causing cascading errors across multiple models.
- **Don't stop the wrong PostgreSQL**: Always confirm which version holds your actual data before
  running `brew services stop`. Stopping the wrong one will make all your databases unreachable.
- **Port conflict cascades**: The PostgreSQL error causes `brew services` to keep retrying (KeepAlive=true
  in the plist), flooding your logs every ~10 seconds. Stopping the conflicting service immediately clears this.
- **`dropdb` fails if sessions are open**: Always terminate active sessions before dropping a database.
  Odoo may keep a connection alive even after shutdown — use `pg_terminate_backend` first.

---

## Verification

```bash
# Verify PostgreSQL is clean
brew services list | grep postgres
lsof -i :5432

# Verify Odoo 18 starts without errors
.venv/bin/python odoo-bin -c odoo18_dev.conf --stop-after-init 2>&1 | grep -E "(ERROR|FATAL|INFO.*version)"

# Verify DB is gone
psql -U odoo -h localhost -l | grep <db_name>
```

---

## References

- Related: `upgrade/analytic-distribution-migration-odoo18.md`
- PostgreSQL Homebrew docs: https://formulae.brew.sh/formula/postgresql@14

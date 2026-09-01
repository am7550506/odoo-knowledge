# CSRF Token / Session Conflict Between Multiple Odoo Instances

| Field         | Value                                      |
|---------------|---------------------------------------------|
| Category      | setup                                       |
| Odoo Versions | All (17, 18, 19)                            |
| Severity      | 🔴 Critical                                 |
| Last Verified | 2026-09-01                                  |
| Author        | ENG/Gamal Mansour                           |

**Tags:** `csrf`, `session`, `multi-instance`, `localhost`, `cookie`, `bad-request`, `400`

---

## Problem

When running two or more Odoo instances simultaneously on the same machine (e.g., Odoo 17 on port 8017 and Odoo 19 on port 8019), the browser randomly shows:

```
Bad Request
Session expired (invalid CSRF token)
```

The error appears even though the user just logged in or did nothing wrong.

## Root Cause

All Odoo instances running on `localhost` share the **same browser cookie namespace** (same domain = `localhost`). The session cookie is named `session_id` by default in all versions.

When both instances are open in the same browser:
1. Browser sends the **same `session_id` cookie** to both ports.
2. Instance A's session is used to validate a request on Instance B.
3. The CSRF token embedded in the session doesn't match → **400 Bad Request**.

Additionally, if no `data_dir` is set, all instances default to `~/.local/share/Odoo/sessions/` (on Linux) or a shared OS-level temp path — further mixing their sessions.

## Solution ✅

### Fix 1 — Set Separate `data_dir` per Instance (Root Fix)

Add `data_dir` to each Odoo config file. This ensures each instance stores its sessions in a completely isolated folder:

**`odoo17_dev.conf`**:
```ini
[options]
http_port = 8017
longpolling_port = 8717
data_dir = /path/to/odoo17.0/data

; Optional: only show v17 databases
db_filter = .*17.*
```

**`odoo19_dev.conf`**:
```ini
[options]
http_port = 8019
longpolling_port = 8719
data_dir = /path/to/odoo19.0/data

; Optional: only show v19 databases
db_filter = .*19.*
```

Then create the directories:
```bash
mkdir -p /path/to/odoo17.0/data/sessions
mkdir -p /path/to/odoo19.0/data/sessions
```

### Fix 2 — Use Different Browser Profiles (Quick Workaround)

If you can't touch the config, use different browser profiles for each Odoo instance. Each profile has its own isolated cookie store.

- Chrome: `chrome://version/` → use Profile picker
- Firefox: Use **Multi-Account Containers** extension

### Fix 3 — Use Different Browsers

One instance in Chrome, the other in Firefox. Not elegant but it works in a pinch.

---

## Why `data_dir` Fixes It

Odoo's session storage (`werkzeug` FileSystemSessionStore) writes session files to:
```
{data_dir}/sessions/
```

Each request validates the CSRF token against **the session file on disk**. With separate `data_dir`, Instance A cannot accidentally read or validate Instance B's sessions.

---

## ⚠️ Pitfalls

- **`xmlrpc_port` is deprecated** — In Odoo 15+, use `http_port` instead. Both work for now but `xmlrpc_port` may cause issues.
  - 🔴 **On Odoo 19.0 specifically, `xmlrpc_port` no longer works at all** — confirmed by
    running a real instance with `xmlrpc_port = 8021` in the conf: no error, no warning
    about the key itself, the server just silently binds to the **default 8069** instead.
    `odoo/tools/config.py` on 19.0 only registers `-p/--http-port` (dest `http_port`) —
    the `xmlrpc_port` CLI alias is gone. If a v19 instance "ignores" the port you set,
    check the conf key name first, not the port number. Use `http_port =` on 19.0.
  - `session_cookie_name` is **not** a real config option on 19.0 either (not referenced
    anywhere in `odoo/`) — it loads with `unknown option ... stored as-is, without
    parsing` and does nothing. It does not achieve session isolation; use `data_dir`
    (Fix 1 above) for that instead.
- **Don't use the same `longpolling_port`** — Default is 8072 for all. Assign unique ones (e.g., 8717, 8719) to avoid bus conflicts.
- **db_filter is regex** — `.*17.*` matches any database containing "17". Adjust to match your naming convention.
- **Restart required** — You must fully restart the Odoo process after changing the config for `data_dir` to take effect.
- **Clear browser cookies** after applying the fix to start fresh.

## Verification

After restarting both instances:

```bash
# Check sessions are stored separately
ls /path/to/odoo17.0/data/sessions/
ls /path/to/odoo19.0/data/sessions/

# Check both ports are listening
lsof -i :8017
lsof -i :8019
```

Open both instances in the **same browser tab** — no more CSRF errors.

## References

- Related: `setup/multi-version-odoo-setup.md`
- Werkzeug Session: https://werkzeug.palletsprojects.com/en/3.x/contrib/sessions/
- Odoo Source: `odoo/http.py` → `FilesystemSessionStore`

# Odoo 19 Server Stuck Down / Registry Load Loop: Started With the Wrong Python Interpreter (missing `xmlsec`)

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | setup |
| Odoo Versions | 19 |
| Severity      | 🔴 Critical |
| Last Verified | 2026-09-22 |
| Author        | ENG/Gamal Mansour |

**Tags:** `venv`, `python`, `xmlsec`, `l10n_nl_reports`, `crash`, `registry`, `dev-server`, `macos`

---

## Problem

`ayadi_test` (project under `odoo19.0`) was unreachable — `ERR_CONNECTION_REFUSED` on
`localhost:8019`. The log showed the server had died hours earlier during Odoo's
dev-mode auto-reload with:

```
Fatal Python error: init_import_site: Failed to import the site module
PermissionError: [Errno 1] Operation not permitted: '.../odoo19.0/.venv/pyvenv.cfg'
```

Restarting it (copying the exact command used for a sibling project on the same
machine, `el-motaheda`) got the port listening again, but the registry then failed to
load in a tight retry loop every ~60s:

```
CRITICAL ayadi_test odoo.modules.module: Couldn't load module l10n_nl_reports
...
ModuleNotFoundError: No module named 'xmlsec'
ERROR ayadi_test odoo.registry: Failed to load registry
```

## Root Cause

Two different Python interpreters exist for the same `odoo19.0` checkout:
- the bare pyenv interpreter: `/Users/macera/.pyenv/versions/3.12.13/bin/python3`
- the project's own venv: `/Users/macera/Documents/Odoo/Odoo Github/odoo19.0/.venv/bin/python3`

`xmlsec` (needed by the Dutch localization, `l10n_nl_reports` — relevant here because
one of `ayadi_test`'s companies is a B.V.) is installed **only** in the project's
`.venv`, not in the bare pyenv interpreter. The sibling project (`el-motaheda`)
happens to run fine on the bare interpreter because it doesn't have a company that
pulls in `l10n_nl_reports`. Copying its launch command for `ayadi_test` silently used
the wrong interpreter — the HTTP port still bound (werkzeug starts before module
loading), which is why `lsof`/`curl` looked "almost fine" while every real request
would have 500'd once the registry tried (and failed) to load.

The original `PermissionError: ... pyvenv.cfg` crash during auto-reload looked
unrelated but was the reason the server was down at all in the first place — it
appears to have been a transient filesystem hiccup (file was normal, correctly owned,
no unusual xattrs, and executed fine on inspection); it did not recur on a plain
restart and is not the actionable part of this entry.

## Solution ✅

Always launch **this project** with its own venv's Python, not the bare pyenv one:

```bash
cd "/Users/macera/Documents/Odoo/Odoo Github/odoo19.0"
"/Users/macera/Documents/Odoo/Odoo Github/odoo19.0/.venv/bin/python3" odoo-bin \
    -c /Users/macera/Documents/ayadi/ayadi_test.conf
```

If a module load failure like this shows up again, check which interpreter actually
has the missing package before assuming the dependency itself needs installing:

```bash
"/Users/macera/Documents/Odoo/Odoo Github/odoo19.0/.venv/bin/python3" -c "import xmlsec"
/Users/macera/.pyenv/versions/3.12.13/bin/python3 -c "import xmlsec"
```

## ⚠️ Pitfalls

- **Don't assume the launch command from a sibling project under the same
  `odoo19.0` folder is safe to reuse.** Two projects sharing one Odoo checkout can
  still need two different interpreters if their installed localizations differ.
- A server that's "up" on `lsof`/`curl` (port bound) is not the same as a server whose
  **registry loaded**. Always grep the log for `Registry loaded` vs. `Failed to load
  registry` after a restart — the HTTP socket opens before any module is imported.
- A registry load failure repeats **every request/cron tick** (roughly once a minute
  here) until fixed — don't mistake the retry cadence for the server being merely
  slow to start.

## Verification

```bash
tail -f /Users/macera/Documents/ayadi/logs/odoo.log | grep -E "Registry loaded|Failed to load registry"
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8019/web/login   # expect 200
```

## References

- Project conf: `/Users/macera/Documents/ayadi/ayadi_test.conf` (`http_port = 8019`)
- `enterprise/l10n_nl_reports/wizard/l10n_nl_reports_sbr_tax_report_wizard.py` (the
  `import xmlsec` that fails on the wrong interpreter)

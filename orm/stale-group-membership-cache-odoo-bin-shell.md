# Group/View Changes Made via `odoo-bin shell` Don't Show in an Already-Running Dev Server

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | orm                                        |
| Odoo Versions | 16, 17, 18, 19                             |
| Severity      | 🟡 Medium                                  |
| Last Verified | 2026-09-05                                 |
| Author        | ENG/Gamal Mansour                          |

**Tags:** `orm`, `ormcache`, `res.groups`, `res.users`, `shell`, `dev-server`, `caching`, `multi-process`

---

## Problem

> Assigned a user to a newly-created `res.groups` record from a separate
> `odoo-bin shell` session (to test a `groups="..."` button/menu), while a
> normal `odoo-bin -c <conf>` dev server was already running in another
> terminal/process for the same database. `psql` confirms the row exists
> in `res_groups_users_rel`, and re-running the exact same check inside a
> *fresh* `odoo-bin shell` also confirms `user.has_group(...)` is `True`.
> But logging into the already-running server in the browser as that same
> user, the button/menu restricted by that group still does not appear —
> `get_views`/`get_view` returns an arch with the group-restricted element
> stripped out, as if the user were not in the group at all.

## Root Cause

> `res.users._get_group_ids()` (which `has_group`/the view's
> `_postprocess_access_rights` group filtering both rely on) is decorated
> `@tools.ormcache('self.id')` — cached in the **Python process's own
> memory**, keyed only by user id. `odoo-bin shell` and a running
> `odoo-bin -c <conf>` server (started without `--workers`, i.e. a single
> threaded process) are two *separate OS processes*, each with its own
> registry and its own copy of this cache. A write to
> `res_groups_users_rel` from the shell process commits fine to Postgres,
> but the already-running server process has no way to know its
> in-memory `_get_group_ids(uid)` result is now stale — there is no
> `--workers`-style registry-signaling polling loop invalidating caches
> between arbitrary one-off script processes and a long-lived dev server;
> that signaling path is for prefork worker processes of the *same*
> `odoo-bin` invocation, not for a separate `shell` invocation.

## Solution ✅

Restart the running dev server process after making any change to
security groups (or anything else touched by an `ormcache`) from a
separate `odoo-bin shell` / one-off script, before testing in the
browser against that same running server:

```bash
# Find and stop the running dev server for this project
ps aux | grep "<project>.conf" | grep -v grep
kill <pid>

# Restart it
cd "{workspace}"
nohup python3 odoo-bin -c /path/to/project.conf > /path/to/odoo.log 2>&1 &
disown
```

A brand-new `odoo-bin shell` process (or any fresh process) always
computes `_get_group_ids` from scratch on first access, so it will
**never** reproduce this staleness — which is exactly what makes it
confusing: the shell "proves" the fix works while the browser, against
the older process, still shows the old behavior.

## ⚠️ Pitfalls

- **A fresh shell check is not proof the running server sees it too.**
  Always verify against the actual server process the browser is
  hitting, not a throwaway `odoo-bin shell` invocation — they do not
  share memory.
- Fetch the actual authenticated `uid` from the browser session (e.g.
  `fetch('/web/session/get_session_info')`) before concluding "the user
  has the group" — do not assume the browser session's user is the one
  you just edited.
- This is unrelated to view *combination* caching (which is correctly
  recomputed via registry signaling on module install/upgrade within
  the same process) — it is specifically the per-user
  `_get_group_ids` / group-membership cache.
- In a real multi-worker production deployment (`--workers N`), this
  self-heals within a few seconds via the registry signaling table that
  Odoo polls per request — this pitfall is specific to ad-hoc local dev
  where a `shell` script and a persistent single dev-server process are
  mixed.

## Verification

```bash
# In the browser (authenticated), confirm both the uid and the
# resolved arch/group state match expectations after restarting:
fetch('/web/session/get_session_info', {method:'POST', headers:{'Content-Type':'application/json'},
  body: JSON.stringify({jsonrpc:"2.0", method:"call", params:{}})})
  .then(r => r.json()).then(j => console.log(j.result.uid, j.result.is_admin));
```

## References

- Related file: `orm/odoo-19-res-groups-category-id-deprecation.md`
- Odoo core: `odoo/addons/base/models/res_users.py::_get_group_ids`
  (`@tools.ormcache('self.id')`)
- Odoo core: `odoo/addons/base/models/ir_ui_view.py::_postprocess_access_rights`

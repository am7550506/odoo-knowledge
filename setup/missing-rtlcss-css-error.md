# Missing rtlcss Causes CSS Error in Arabic Layout

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | setup                                      |
| Odoo Versions | All                                        |
| Severity      | 🔴 Critical                                |
| Last Verified | 2026-05-21                                 |
| Author        | ENG/Mohamed Hamdy                          |

**Tags:** `rtlcss`, `css`, `arabic`, `ui`, `setup`, `nodejs`

---

## Problem

> When opening a database with an Arabic (or any Right-To-Left) interface, the UI layout breaks completely and a red warning bar appears at the bottom of the screen.

```
A css error occured, using an old style to render this page
```

## Root Cause

> Odoo uses `rtlcss` to dynamically convert standard LTR CSS into RTL CSS for Arabic layouts. If `rtlcss` is missing from the system, the CSS compilation fails. Without the required CSS assets, Odoo falls back to a broken, unstyled HTML layout and displays the "old style" error warning.

## Solution ✅

> Install `rtlcss` globally via `npm` (Node Package Manager).

```bash
# Ensure npm is installed (Ubuntu/Debian)
sudo apt update
sudo apt install nodejs npm

# Install rtlcss globally
sudo npm install -g rtlcss
```

> After installing, you must restart your Odoo server. If the issue persists, clear your browser cache or force Odoo to regenerate assets:

```bash
# Start Odoo with -u all to trigger asset recompilation (optional if simple restart fails)
./odoo-bin -d YOUR_DATABASE -u all
```

## ⚠️ Pitfalls

- Forgetting the `-g` flag: `npm install rtlcss` without `-g` installs it only in the current directory. Odoo needs it globally.
- Permission issues: Global npm installs usually require `sudo`.
- Stale Assets: Sometimes simply installing `rtlcss` isn't enough; the corrupted asset bundles might remain cached in the database. In that case, deleting records from `ir_attachment` where `name` like `%web.assets%` might be necessary, but usually `-u all` or clicking "Regenerate Assets Bundles" (if you can reach Developer Mode) works.

## Verification

> Check if `rtlcss` is accessible globally:

```bash
rtlcss --version
```

It should output the installed version (e.g., `2.5.0` or higher).

## References

- [Odoo Documentation: System Dependencies](https://www.odoo.com/documentation/17.0/administration/install/install.html#dependencies)

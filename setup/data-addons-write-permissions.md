# Data Addons Directory Write Permission Issues (Assets Loading Error)

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | setup                                      |
| Odoo Versions | All                                        |
| Severity      | 🔴 Critical                                |
| Last Verified | 2026-05-30                                 |
| Author        | ENG/Gamal Mansour                          |

**Tags:** `setup`, `permissions`, `data_dir`, `assets`, `AssetsLoadingError`

---

## Problem

When loading the Odoo backend, the UI crashes with an Owl lifecycle error caused by `AssetsLoadingError`:
```
Caused by: AssetsLoadingError: The loading of /web/assets/.../web_editor.backend_assets_wysiwyg.min.js failed
```
Refreshing the browser does not resolve the issue.

## Root Cause

Odoo compiles JS and CSS files and attempts to write them to the local filesystem under the path `<data_dir>/addons/<version>/`. If this folder has incorrect permissions (for example, `dr-x------` where the owner has no write permission `w`), Odoo fails to write the compiled bundles to disk. As a result, the browser receives a 404 or fails to load the asset files.

## Solution ✅

Modify the permissions of the `<data_dir>/addons` directory recursively to grant write permissions to the user running the Odoo service:

```bash
chmod -R 755 /path/to/data_dir/addons
```

For example, on macOS/Linux:
```bash
chmod -R 755 /Users/gamal/odoo/odoo17.0/data/addons
```

## ⚠️ Pitfalls

- **Incorrect Owner**: Sometimes the folder is created by `root` or another user during setup, which prevents the Odoo user from writing to it. In that case, change the ownership first:
  ```bash
  chown -R odoo_user:odoo_group /path/to/data_dir
  ```

## Verification

To verify:
1. Fix the permissions using `chmod`.
2. Clear the browser cache or perform a hard refresh (`Cmd+Shift+R` or `Ctrl+F5`).
3. Confirm that the UI loads and new files are generated under `<data_dir>/addons/<version>/`.

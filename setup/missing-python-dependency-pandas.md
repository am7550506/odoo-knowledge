# Missing Python Dependency Pandas Causes Odoo Registry Load Failure

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | setup                                      |
| Odoo Versions | All                                        |
| Severity      | 🔴 Critical                                |
| Last Verified | 2026-05-21                                 |
| Author        | ENG/Mohamed Saber                          |

**Tags:** `pandas`, `dependency`, `pip`, `setup`, `registry`, `hrms_dashboard`

---

## Problem

When starting the Odoo server or trying to load a database, the server crashes during registry loading with `ModuleNotFoundError: No module named 'pandas'`. This prevents Odoo from booting up and serving requests.

```
2026-05-21 13:37:18,728 8542 CRITICAL ? odoo.modules.module: Couldn't load module hrms_dashboard 
2026-05-21 13:37:18,730 8542 ERROR ? odoo.registry: Failed to load registry 
...
  File "/home/hamdy/odoo/odoo19/Zakham/hrms_dashboard/models/hr_employee.py", line 23, in <module>
    import pandas as pd
ModuleNotFoundError: No module named 'pandas'
```

## Root Cause

Custom or community modules (such as `hrms_dashboard`) import the `pandas` library for advanced data manipulation or grouping (e.g., in analytics or dashboard charts). If `pandas` is not installed in the python virtual environment where Odoo is running, the top-level import crashes during the initial import scan of Odoo modules.

## Solution ✅

Install `pandas` package in Odoo's active virtual environment (`venv`).

```bash
# 1. Activate the Odoo project virtual environment
source /home/hamdy/odoo/odoo19/venv/bin/activate

# 2. Install pandas inside the virtual environment
/home/hamdy/odoo/odoo19/venv/bin/pip install pandas
```

Also, ensure the custom module specifies `pandas` under `external_dependencies` in its `__manifest__.py` file:

```python
    'external_dependencies': {
        'python': ['pandas'],
    },
```

## ⚠️ Pitfalls

- **Incorrect Virtual Environment:** Installing `pandas` via global `pip install pandas` or `sudo pip install pandas` may install it in the system python environment, while Odoo is configured to run inside a dedicated virtualenv (`/home/hamdy/odoo/odoo19/venv`). Always make sure to use the absolute path to the virtualenv pip or activate it first.
- **Transitive Dependencies:** `pandas` will also pull `numpy` and other packages, which can take a few seconds to download and build.

## Verification

To verify that Odoo loads the registry and starts up properly without pandas-related crashes, run a dry-run test against the database:

```bash
/home/hamdy/odoo/odoo19/venv/bin/python ./odoo-bin --addons-path=/home/hamdy/odoo/odoo19/addons,/home/hamdy/odoo/odoo19/enterprise,/home/hamdy/odoo/odoo19/Zakham -d Zakham --stop-after-init
```

If it prints `Registry loaded in ...` and exits cleanly with exit code 0, the problem is fully resolved.

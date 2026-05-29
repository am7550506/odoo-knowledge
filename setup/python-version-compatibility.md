# Python Version Compatibility Per Odoo Version

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | setup                                      |
| Odoo Versions | All (15, 16, 17, 18, 19)                   |
| Severity      | 🔴 Critical                                |
| Last Verified | 2026-05-20                                 |
| Author        | ENG/Mohamed Saber                          |

**Tags:** `python`, `version`, `venv`, `compatibility`, `release.py`

---

## Problem

Using a Python version outside the supported range for an Odoo version causes cryptic import errors, syntax errors, or startup failures. The system Python may not match what Odoo requires.

```
# Example: Using Python 3.14 with Odoo 15
SyntaxError: invalid syntax
# or
ImportError: cannot import name 'X' from 'collections'
```

## Root Cause

Each Odoo version supports a specific range of Python versions. This is defined in `odoo/release.py` as `MIN_PY_VERSION` and `MAX_PY_VERSION`. Older Odoo versions removed deprecated Python features (like `collections.MutableMapping` moved to `collections.abc`) that newer Python versions enforce.

## Solution ✅

### Step 1: Check the supported range

```bash
grep -E "MIN_PY|MAX_PY" /path/to/odoo/release.py
```

### Step 2: Known compatibility matrix

| Odoo Version | Min Python | Max Python | Recommended | Notes |
|:---:|:---:|:---:|:---:|:---|
| 15.0 | 3.7 | 3.10 | 3.10 | No MIN/MAX in release.py, check setup.py |
| 16.0 | 3.7 | 3.11 | 3.10 | No MIN/MAX in release.py, check setup.py |
| 17.0 | 3.10 | 3.12 | 3.12 | First version with MIN/MAX in release.py |
| 18.0 | 3.10 | 3.13 | 3.12 | |
| 19.0 | 3.10 | 3.14 | 3.14 | Latest, supports newest Python |

> **Note:** Odoo 15 and 16 do NOT have `MIN_PY_VERSION`/`MAX_PY_VERSION` in `release.py`. Check `setup.py` or `setup.cfg` for `python_requires` instead.

### Step 3: Install the correct Python version

```bash
# If system Python is too new (e.g., 3.14 but Odoo 15 needs 3.10)
sudo apt install python3.10 python3.10-venv python3.10-dev

# Create venv with the correct version
python3.10 -m venv /home/hamdy/odoo/odoo15/venv
```

### Step 4: Verified working combinations on our system

| Odoo | Python in venv | Status |
|:---:|:---:|:---:|
| 15 | 3.10.20 | ✅ Working |
| 16 | 3.11.15 | ✅ Working |
| 17 | 3.12.13 | ✅ Working |
| 18 | 3.14.4 | ✅ Working |
| 19 | 3.14.4 | ✅ Working |

## ⚠️ Pitfalls

- **Never assume system Python works** — Always check `release.py` or `setup.py` first.
- **Older Python versions may need manual installation** — `sudo apt install python3.10 python3.10-venv python3.10-dev`
- **Don't forget `-dev` package** — Without `python3.X-dev`, C extensions (psycopg2, lxml, gevent) won't compile.
- **Each venv is tied to its Python version** — If you upgrade the system Python, existing venvs may break. Recreate them.

## Verification

```bash
# Check venv Python version
/path/to/venv/bin/python --version

# Verify it's within the supported range
/path/to/venv/bin/python -c "
import sys
print(f'Python {sys.version}')
try:
    from odoo.release import MIN_PY_VERSION, MAX_PY_VERSION
    v = sys.version_info[:2]
    assert MIN_PY_VERSION <= v <= MAX_PY_VERSION, f'{v} not in [{MIN_PY_VERSION}, {MAX_PY_VERSION}]'
    print('✅ Python version OK')
except ImportError:
    print('⚠️  No MIN/MAX in release.py — check setup.py manually')
"
```

## References

- `odoo/release.py` in each version
- `setup.py` / `setup.cfg` for older versions (15, 16)

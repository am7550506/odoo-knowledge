# Multi-Version Odoo Setup — venv & Database Strategy

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | setup                                      |
| Odoo Versions | All (15, 16, 17, 18, 19)                   |
| Severity      | 🟡 Medium                                  |
| Last Verified | 2026-05-20                                 |
| Author        | ENG/Mohamed Saber                          |

**Tags:** `venv`, `database`, `multi-version`, `postgresql`, `ports`, `setup`

---

## Problem

Running multiple Odoo versions on the same machine requires careful isolation of Python environments, databases, and server ports to avoid conflicts.

## Root Cause

All Odoo versions share the same system-level services (PostgreSQL, wkhtmltopdf) but need different Python packages (and sometimes different Python versions). Without isolation, installing packages for one version can break another.

## Solution ✅

### Folder Structure

```
/home/hamdy/odoo/
├── odoo-knowledge/           ← Shared knowledge base (this repo)
├── odoo_installation_notes.md
├── odoo15/
│   ├── addons/               ← Community modules
│   ├── enterprise/           ← Enterprise modules
│   ├── odoo/                 ← Core (odoo/addons = base modules)
│   ├── odoo-bin              ← Entry point
│   ├── requirements.txt
│   └── venv/                 ← Isolated Python venv (Python 3.10)
├── odoo16/
│   └── venv/                 ← Python 3.11
├── odoo17/
│   └── venv/                 ← Python 3.12
├── odoo18/
│   └── venv/                 ← Python 3.14
└── odoo19/
    └── venv/                 ← Python 3.14
```

### Port Assignment (avoid conflicts)

| Version | HTTP Port | Longpolling Port |
|:---:|:---:|:---:|
| Odoo 15 | 8015 | 8072 |
| Odoo 16 | 8016 | 8073 |
| Odoo 17 | 8017 | 8074 |
| Odoo 18 | 8018 | 8075 |
| Odoo 19 | 8069 (default) | 8072 (default) |

### Database Naming

One database per Odoo version:
- `odoo15`, `odoo16`, `odoo17`, `odoo18`, `odoo19`

### PostgreSQL Setup (one-time)

```bash
sudo apt install -y postgresql postgresql-client
sudo -u postgres createuser -s $(whoami)
```

### Full Init Sequence for Any Version

```bash
VERSION=19  # Change this

# 1. Check supported Python
grep -E "MIN_PY|MAX_PY" /home/hamdy/odoo/odoo${VERSION}/odoo/release.py

# 2. Create venv (use correct Python version!)
python3.XX -m venv /home/hamdy/odoo/odoo${VERSION}/venv

# 3. Upgrade pip
/home/hamdy/odoo/odoo${VERSION}/venv/bin/pip install --upgrade pip setuptools wheel

# 4. Install lxml separately (if Python >= 3.13, see lxml-build-failure-python313-plus.md)
/home/hamdy/odoo/odoo${VERSION}/venv/bin/pip install lxml lxml-html-clean

# 5. Install requirements (skip lxml)
grep -v "^lxml" /home/hamdy/odoo/odoo${VERSION}/requirements.txt > /tmp/req_nolxml.txt
/home/hamdy/odoo/odoo${VERSION}/venv/bin/pip install -r /tmp/req_nolxml.txt

# 6. Init database
/home/hamdy/odoo/odoo${VERSION}/venv/bin/python /home/hamdy/odoo/odoo${VERSION}/odoo-bin \
    --addons-path=/home/hamdy/odoo/odoo${VERSION}/addons,/home/hamdy/odoo/odoo${VERSION}/enterprise,/home/hamdy/odoo/odoo${VERSION}/odoo/addons \
    -d odoo${VERSION} -i base --stop-after-init

# 7. Start server
/home/hamdy/odoo/odoo${VERSION}/venv/bin/python /home/hamdy/odoo/odoo${VERSION}/odoo-bin \
    --addons-path=/home/hamdy/odoo/odoo${VERSION}/addons,/home/hamdy/odoo/odoo${VERSION}/enterprise,/home/hamdy/odoo/odoo${VERSION}/odoo/addons \
    -d odoo${VERSION} --http-port=80${VERSION}
```

### VS Code / Antigravity Auto-Activate venv

Each project has `.vscode/terminal_init.sh` + `settings.json` configured to auto-activate the venv when opening a terminal. See `setup/vscode-auto-activate-venv.md`.

## ⚠️ Pitfalls

- **Never run two Odoo instances on the same port** — Check with `lsof -i :8069` before starting.
- **venvs are NOT portable** — If you move the odoo folder, recreate the venvs.
- **Don't mix pip installs** — Always use the full path to the venv's pip, never bare `pip`.
- **PostgreSQL user must be superuser** — `createuser -s` is important, Odoo needs it for DB creation.

## Verification

```bash
# Check which Odoo instances are running
ps aux | grep odoo-bin | grep -v grep

# Check port usage
ss -tlnp | grep -E "80(15|16|17|18|69)"

# Test HTTP response
curl -s -o /dev/null -w "%{http_code}" http://localhost:8069/web/login
```

## References

- Related: `setup/python-version-compatibility.md`
- Related: `setup/lxml-build-failure-python313-plus.md`
- Related: `setup/system-dependencies-ubuntu.md`

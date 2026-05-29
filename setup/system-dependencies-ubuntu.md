# System Dependencies for Odoo on Ubuntu

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | setup                                      |
| Odoo Versions | All (15, 16, 17, 18, 19)                   |
| Severity      | 🔴 Critical                                |
| Last Verified | 2026-05-20                                 |
| Author        | ENG/Mohamed Saber                          |

**Tags:** `apt`, `dependencies`, `ubuntu`, `libpq`, `libldap`, `build-essential`, `npm`

---

## Problem

`pip install -r requirements.txt` fails with various compilation errors when system development libraries are missing.

Common errors:
```
# Missing libpq-dev
Error: pg_config executable not found

# Missing libldap2-dev
fatal error: lber.h: No such file or directory

# Missing libxml2-dev
fatal error: libxml/xmlversion.h: No such file or directory

# Missing libjpeg-dev
fatal error: jpeglib.h: No such file or directory
```

## Root Cause

Many Python packages in Odoo's requirements (psycopg2, python-ldap, lxml, Pillow, etc.) are C extensions that need system-level development headers to compile from source.

## Solution ✅

Install ALL required system packages **BEFORE** creating the venv or running pip:

```bash
sudo apt update && sudo apt install -y \
    postgresql postgresql-client libpq-dev \
    python3-dev python3-venv build-essential \
    libldap2-dev libsasl2-dev libssl-dev \
    libxml2-dev libxslt1-dev \
    libjpeg-dev zlib1g-dev libfreetype-dev \
    libmagic1t64 \
    node-less npm
```

### Package → Why It's Needed

| Package | Required By | Purpose |
|---|---|---|
| `postgresql`, `postgresql-client` | Odoo core | Database server |
| `libpq-dev` | `psycopg2` | PostgreSQL C client headers |
| `python3-dev` | All C extensions | Python development headers |
| `build-essential` | All C extensions | gcc, make, etc. |
| `libldap2-dev`, `libsasl2-dev` | `python-ldap` | LDAP authentication |
| `libssl-dev` | `cryptography` | SSL/TLS support |
| `libxml2-dev`, `libxslt1-dev` | `lxml` | XML processing |
| `libjpeg-dev`, `zlib1g-dev`, `libfreetype-dev` | `Pillow` | Image processing |
| `libmagic1t64` | `python-magic` | File type detection |
| `node-less`, `npm` | Odoo assets | LESS/SCSS compilation |

## ⚠️ Pitfalls

- **On Ubuntu 24.04+ (Noble/Resolute):** The package is `libmagic1t64`, NOT `libmagic1` (64-bit time_t transition).
- **Install BEFORE creating venv** — Some packages like `python3-dev` need to match the Python version.
- **If using a non-system Python (e.g., python3.10):** You need `python3.10-dev` specifically, not just `python3-dev`.
- **`node-less` is needed mainly for Odoo 15/16** — Newer versions use SCSS, but it doesn't hurt to have it.

## Verification

```bash
# Quick check that key headers exist
dpkg -l | grep -E "libpq-dev|libldap2-dev|libxml2-dev|libjpeg-dev|python3-dev" | wc -l
# Should be >= 5
```

## References

- Odoo official install docs: https://www.odoo.com/documentation/19.0/administration/install/source.html
- Related: `setup/wkhtmltopdf-not-in-ubuntu-repos.md`

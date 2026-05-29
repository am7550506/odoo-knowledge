# lxml Build Failure on Python 3.13+ / Ubuntu 24.04+

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | setup                                      |
| Odoo Versions | 17, 18, 19                                 |
| Severity      | 🔴 Critical                                |
| Last Verified | 2026-05-20                                 |
| Author        | ENG/Mohamed Saber                          |

**Tags:** `lxml`, `python`, `build`, `compilation`, `pip`, `gcc`, `ubuntu`

---

## Problem

When running `pip install -r requirements.txt`, the `lxml` package fails to build from source on Python 3.13+ with Ubuntu 24.04+.

```
error: command '/usr/bin/x86_64-linux-gnu-gcc' failed with exit code 1
src/lxml/etree.c: In function '__pyx_f_4lxml_5etree_fixThreadDictNamesForDtd':
src/lxml/etree.c:28042:49: error: passing argument 1 of '__pyx_f_4lxml_5etree__fixThreadDictPtr'
    from incompatible pointer type [-Wincompatible-pointer-types]
```

## Root Cause

Odoo's `requirements.txt` pins `lxml==5.2.1` for `python_version >= '3.12'`. This version's C code has **pointer type incompatibilities** with newer `libxml2` headers shipped in Ubuntu 24.04+ (libxml2 2.15+). The pinned version was compiled against older headers and cannot be built on newer systems.

## Solution ✅

**2-step approach:** Install latest lxml first, then install the rest without lxml.

```bash
# Step 1: Install latest lxml (has Python 3.13+/3.14 support)
/path/to/venv/bin/pip install lxml

# Step 2: Install lxml-html-clean (required by Odoo 17+)
/path/to/venv/bin/pip install lxml-html-clean

# Step 3: Install requirements WITHOUT lxml lines
grep -v "^lxml" requirements.txt > /tmp/requirements_nolxml.txt
/path/to/venv/bin/pip install -r /tmp/requirements_nolxml.txt
rm /tmp/requirements_nolxml.txt
```

## ⚠️ Pitfalls

- **Do NOT try to force-install the pinned lxml version** — it will always fail on Python 3.13+/3.14 with new libxml2.
- **lxml-html-clean is a separate package** — it was split from lxml starting from lxml 5.2+. Odoo 17+ needs it explicitly.
- **The grep command must use `^lxml`** (with `^`) to only skip lines STARTING with `lxml`, not lines that happen to contain the word elsewhere.
- **Order matters** — Install lxml FIRST, then the rest. If you install requirements.txt first, pip will fail on lxml and skip everything after it.

## Verification

```bash
/path/to/venv/bin/python -c "import lxml; from lxml.html.clean import Cleaner; print(f'lxml {lxml.__version__} OK')"
```

## References

- Odoo requirements.txt version markers: lines 36-39
- lxml changelog: https://lxml.de/changes.html

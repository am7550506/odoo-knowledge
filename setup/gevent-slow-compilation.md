# gevent / greenlet Long Compilation on New Python Versions

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | setup                                      |
| Odoo Versions | All (15, 16, 17, 18, 19)                   |
| Severity      | 🟢 Low                                     |
| Last Verified | 2026-05-20                                 |
| Author        | ENG/Mohamed Saber                          |

**Tags:** `gevent`, `greenlet`, `compilation`, `pip`, `slow`, `build`

---

## Problem

When running `pip install -r requirements.txt`, the `gevent` and `greenlet` packages take an unusually long time (2-5 minutes) to install, appearing to hang.

```
Building wheel for gevent (pyproject.toml) ... -\|/-\|/-\|/
```

## Root Cause

For newer Python versions (3.13, 3.14), there are **no pre-built wheel packages** available on PyPI. pip falls back to building from source, which involves compiling a significant amount of C code (libev, c-ares, libuv wrappers).

This is **normal behavior**, not an error.

## Solution ✅

**Just wait.** This is expected. The build takes 2-5 minutes depending on CPU speed.

To confirm it's actually working (not stuck):
```bash
# In another terminal, check if gcc is running
ps aux | grep gcc | grep -v grep
```

If you want faster installs in the future, you can cache the built wheel:
```bash
# Build wheel once and cache it
pip wheel gevent greenlet -w /home/hamdy/odoo/.pip-cache/

# Use cached wheel in future installs
pip install --find-links=/home/hamdy/odoo/.pip-cache/ gevent greenlet
```

## ⚠️ Pitfalls

- **Do NOT kill the process** — It looks stuck but it's compiling. Killing it will leave a partial install.
- **`build-essential` must be installed** — Without gcc, it will fail (not just be slow).
- **`python3-dev` (or `python3.X-dev`) must be installed** — Without Python headers, compilation fails.

## Verification

```bash
/path/to/venv/bin/python -c "import gevent; import greenlet; print(f'gevent {gevent.__version__}, greenlet {greenlet.__version__} OK')"
```

## References

- gevent PyPI: https://pypi.org/project/gevent/
- Related: `setup/system-dependencies-ubuntu.md`

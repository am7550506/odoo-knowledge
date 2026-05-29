# VS Code / Antigravity Auto-Activate venv in Terminal

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | setup                                      |
| Odoo Versions | All (15, 16, 17, 18, 19)                   |
| Severity      | 🟢 Low                                     |
| Last Verified | 2026-05-20                                 |
| Author        | ENG/Mohamed Saber                          |

**Tags:** `vscode`, `antigravity`, `terminal`, `venv`, `auto-activate`, `settings.json`

---

## Problem

When opening a terminal in Antigravity (VS Code-based), the Python virtual environment is not automatically activated. You have to manually type `source venv/bin/activate` every time.

## Root Cause

The `python.terminal.activateEnvironment` setting requires the Python extension (ms-python.python) to be installed and working. In Antigravity, this extension may not be active, so the setting is ignored.

## Solution ✅

Use a custom terminal profile with `--rcfile` that sources both `.bashrc` and the venv activate script.

### Step 1: Create `.vscode/terminal_init.sh`

```bash
#!/bin/bash
# Source the default bashrc first (for colors, prompt, aliases, etc.)
[ -f ~/.bashrc ] && source ~/.bashrc
# Auto-activate the project's venv
source /home/hamdy/odoo/odoo{VERSION}/venv/bin/activate
```

### Step 2: Update `.vscode/settings.json`

```json
{
    "python.languageServer": "None",
    "python.defaultInterpreterPath": "${workspaceFolder}/venv/bin/python",
    "terminal.integrated.defaultProfile.linux": "bash (venv)",
    "terminal.integrated.profiles.linux": {
        "bash (venv)": {
            "path": "bash",
            "args": ["--rcfile", "${workspaceFolder}/.vscode/terminal_init.sh"]
        }
    }
}
```

### Step 3: Close and reopen terminal

After saving both files, close all existing terminals and open a new one. You should see:
```
(venv) hamdy@hamdy:~/odoo/odoo19$
```

## ⚠️ Pitfalls

- **`terminal_init.sh` uses an ABSOLUTE path** to the venv activate script — `${workspaceFolder}` is NOT expanded inside shell scripts, only in `settings.json`.
- **The `--rcfile` flag replaces `.bashrc`** — That's why we explicitly source `~/.bashrc` in the init script, otherwise you lose colors and aliases.
- **`python.terminal.activateEnvironment: true` does NOT work** without the Python extension — Don't rely on it in Antigravity.
- **Existing terminals won't be affected** — Only new terminals pick up the profile change.

## Verification

Open a new terminal and check:
```bash
# Should show (venv) in prompt
echo $VIRTUAL_ENV
# Should print: /home/hamdy/odoo/odoo{VERSION}/venv

which python
# Should print: /home/hamdy/odoo/odoo{VERSION}/venv/bin/python
```

## References

- VS Code docs: https://code.visualstudio.com/docs/terminal/profiles
- Related: `setup/multi-version-odoo-setup.md`

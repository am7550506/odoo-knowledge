# Odoo.sh Setup Automator GUI Risks & Pre-mortem

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | setup                                      |
| Odoo Versions | All                                        |
| Severity      | 🔴 Critical                                 |
| Last Verified | 2026-05-21                                 |
| Author        | ENG/Mohamed Hamdy                          |

**Tags:** `automation`, `setup`, `ssh`, `database`, `rsync`, `risks`

---

## Problem

> This document serves as a **Pre-mortem Risk Analysis** for the custom GUI tool `Odoo Setup Automator` used internally by the team to clone Odoo.sh staging environments. We assume the tool has been used for 6 months and a critical failure occurred. What went wrong and how do we prevent/mitigate it?

## Root Causes of Future Failures (6-Month Simulation)

### Risk 1: Accidental Production/Staging Overwrite (Catastrophic Data Loss)
**Scenario:** A developer accidentally types an existing, important local database name (e.g., `prod_copy_final`) into the "Target Local DB Name" field.
**Why it happens:** The script runs `sudo -u postgres dropdb --if-exists <local_db>` without asking for a secondary explicit confirmation for the deletion of the existing local database.
**Mitigation:** The tool currently assumes the developer knows what they are typing. *Future improvement:* Add a safety check in the GUI: if `psql -l` shows the database exists, prompt a `messagebox.askyesno` warning the user that the database will be wiped permanently.

### Risk 2: `rsync` Hangs on SSH Key Expiration/Prompt
**Scenario:** A developer attempts to sync the filestore, but their SSH key was revoked on Odoo.sh or they are on a new machine.
**Why it happens:** `rsync` is run via `subprocess.Popen`. If `rsync` is waiting for an interactive password prompt (because public key auth failed), it will freeze the Python thread indefinitely because we aren't handling interactive TTY prompts.
**Mitigation:** The `README.md` explicitly states the SSH key requirement. *Future improvement:* Add a connection timeout or pass `-o BatchMode=yes` to the SSH/rsync command so it fails immediately instead of hanging.

### Risk 3: Corrupt Local Database due to Disk Space Exhaustion
**Scenario:** The developer runs a sync for a 50GB Odoo.sh database, but their local machine only has 10GB free.
**Why it happens:** `pg_dump` streams the data into `psql`, filling the disk. Once the disk is 100% full, the pipe breaks and PostgreSQL crashes or corrupts the partially imported database.
**Mitigation:** Always monitor local disk space before syncing massive enterprise databases. *Future improvement:* Add an OS disk space check via `df -h` in Python before executing the database stream.

### Risk 4: Git Pull Fails due to Uncommitted Changes
**Scenario:** A developer makes local tweaks, doesn't commit them, and runs the sync.
**Why it happens:** The script runs `git fetch --all && git pull`. Git will abort the pull due to uncommitted changes, causing the Code Sync step to fail.
**Mitigation:** The developer must `git stash` or `git commit` before running the tool. The tool logs the error in the console.

### Risk 5: Database Streaming fails with `pg_settings` error
**Scenario:** The developer tries to stream a database from an Odoo.sh staging branch using SSH.
**Why it happens:** Odoo.sh implemented a security patch for CVE-2024-7348 in PostgreSQL 16+. This patch prevents non-superusers from running `pg_dump` over SSH by restricting access to `pg_settings`.
**Mitigation:** The tool was upgraded to automatically pull daily backups from `/home/odoo/backup.daily/` for Production branches. For Staging branches, the developer MUST download the ZIP backup from the Odoo.sh dashboard and provide it to the tool via the new "Backup ZIP Path" field.

## Solution ✅

> Current Workflow Recommendations for the Team:

1. **Always verify the Target Local DB name.** Do not reuse names of databases you wish to keep.
2. **Ensure your SSH Public Key is added to Odoo.sh** before running the tool. Test it manually once via `ssh <user>@<host>` to accept the host fingerprint.
3. **Keep local branches clean** before syncing code.

## ⚠️ Pitfalls

- Running the installer script as root `sudo ./install.sh`. This is wrong. It should be run as the normal user so the `.desktop` file and configurations are placed in the correct `~/.local` user directory.
- Not adding the Odoo.sh host to `~/.ssh/known_hosts` beforehand. The tool may hang if SSH asks "Are you sure you want to continue connecting?".

## Verification

To verify your environment is ready for the tool:
```bash
# Test SSH connectivity and host acceptance
ssh <odoo.sh-username>@<project>.odoo.com "echo SSH is working"
```

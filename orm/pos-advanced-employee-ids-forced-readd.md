# POS Advanced Employee IDs — Forced Re-addition of Administrators

| Field         | Value                                      |
|---------------|---------------------------------------------|
| Category      | orm                                         |
| Odoo Versions | 18, 19                                      |
| Severity      | 🟡 Medium                                   |
| Last Verified | 2026-05-24                                  |
| Author        | ENG/Mohamed Saber                           |

**Tags:** `pos`, `pos_hr`, `advanced_employee_ids`, `many2many`, `write-override`, `sql`

---

## Problem

> When removing a POS Administrator employee from the `advanced_employee_ids`
> ("Advanced rights") field in POS Configuration or Settings, the change is
> silently ignored. After saving, the removed employee reappears in the list.

This happens because:
- `pos_hr/models/pos_config.py` `write()` method blindly appends ALL POS
  Manager group employees via `(4, emp_id)` on **every write**, regardless
  of user intent.
- `pos_hr/models/res_config_settings.py` `create()` method does the same
  thing when saving through the Settings UI.

## Root Cause

In `pos_hr`, the `write()` method on `pos.config` contains:

```python
def write(self, vals):
    if 'advanced_employee_ids' not in vals:
        vals['advanced_employee_ids'] = []
    vals['advanced_employee_ids'] += [(4, emp_id) for emp_id in self._get_group_pos_manager().users.employee_id.ids]
    return super().write(vals)
```

This code runs on **every** write to `pos.config`, even when the user
explicitly removed a POS administrator from the field. The `(4, id)` command
re-links the employee, effectively undoing the user's removal.

The same pattern exists in `res.config.settings.create()`.

## Solution ✅

Create a custom module that:

1. **Calculates "Intended State"**: Before calling `super()`, calculate exactly what the `advanced_employee_ids` field *should* contain based on the commands (`vals`) and current state. For example, if a `(6, 0, ids)` command is sent, the intended state is exactly `ids`. If no commands are sent for that field, the intended state is the current state.
2. **Calls `super()`**: Let `pos_hr` do its normal re-addition. It will append `(4, id)` for ALL managers.
3. **Uses direct SQL** to delete the M2M join table rows for ANY manager who is NOT in the calculated intended state:

```python
intended = intended_states[rec.id]
to_remove = manager_emp_ids - intended

if to_remove:
    self.env.cr.execute(
        """
        DELETE FROM pos_hr_advanced_employee_hr_employee
        WHERE pos_config_id = %s
          AND hr_employee_id IN %s
        """,
        (rec.id, tuple(to_remove))
    )
    self.invalidate_recordset(['advanced_employee_ids'])
```

4. **Same approach for `res.config.settings.create()`** as it suffers from the exact same injection behavior.

### Why SQL?

Using ORM `write()` to remove the employees after `super()` would trigger
`pos_hr`'s `write()` again, which would re-add the managers in an infinite
loop. Direct SQL on the join table is the only way to bypass all ORM
write overrides.

## ⚠️ Pitfalls

- **ORM Cache**: After SQL DELETE, you MUST call `invalidate_recordset()`
  to flush the ORM's in-memory cache, otherwise stale data will be returned.
- **MRO Order**: Our module must depend on `pos_hr` so our write() runs
  BEFORE `pos_hr`'s write() in the MRO chain.
- **Set Command (6, 0, ids)**: Don't forget to handle this command type.
  When the user sets a new list, anything NOT in the list is an implicit
  removal.
- **Settings UI Flow**: The `point_of_sale` settings `create()` strips
  `pos_*` fields from vals and writes them to `pos.config` separately
  via `pos_config.with_context(from_settings_view=True).write()`. So the
  `pos_config.write()` fix covers both direct and settings UI paths.

## Verification

1. Install the module `pos_advanced_rights_fix`
2. Go to POS Configuration → select a POS → edit Advanced rights
3. Remove a user who has POS Administrator access
4. Save → verify the user stays removed
5. Run tests: `python odoo-bin -d DB --test-enable --test-tags pos_advanced_rights_fix`

## References

- Module: `WAMA-Group/pos_advanced_rights_fix/`
- Core file: `addons/pos_hr/models/pos_config.py` (line 17-20)
- Core file: `addons/pos_hr/models/res_config_settings.py` (line 15-20)
- M2M join table: `pos_hr_advanced_employee_hr_employee`

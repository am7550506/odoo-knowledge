# Safely Extending and Modifying Selection Fields in Odoo

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | orm                                        |
| Odoo Versions | All                                        |
| Severity      | 🟡 Medium                                  |
| Last Verified | 2026-05-20                                 |
| Author        | ENG/Mohamed Hamdy                          |

**Tags:** `orm`, `selection`, `fields`, `inheritance`, `selection_add`

---

## Problem

> Modifying or extending selection fields (dropdowns) in Odoo incorrectly can lead to data loss, registry initialization failures, or breaking other modules that rely on the original choices.

## Root Cause

Selection fields in Odoo have two methods of extension depending on ownership of the model:
1. **Modifying direct definition:** If the field is defined in a custom module *you own*, you can directly update the `selection=[...]` list.
2. **Inheriting core/other models:** If the field belongs to a model from another module, you MUST use inheritance with `selection_add` rather than redefining the field entirely, as redefining it will overwrite existing options and break other dependent modules.

## Solution ✅

### Scenario A: Modifying Selection in an Add-on You Own

Update the field definition directly in the Python model:

```python
state = fields.Selection(string="Status/Result", selection=[
    ('apologise', 'Apologise'),
    ('cancelled', 'Cancelled'),
    ('no_reply', 'No Reply'),  # Added selection option
    ('study', 'Study'),
], default='study', copy=True, tracking=True)
```

### Scenario B: Extending Selection in a Model from Another Module

Use Odoo's `selection_add` attribute to append options without breaking existing ones:

```python
class ProjectProject(models.Model):
    _inherit = 'project.project'

    state = fields.Selection(selection_add=[
        ('no_reply', 'No Reply')
    ])
```

## ⚠️ Pitfalls

- **Avoid Redefining Fields:** Redefining a selection field in inheritance *without* `selection_add` will purge all original choices from the field.
- **Translation Missing:** New selection choices added in Python will not show up in translation templates (`.pot`) until you regenerate the translations, and must be manually added to `.po` files if localized.
- **UI Constraints:** If the field's state values are used in XML views (e.g., `invisible="state != 'study'"`), ensure any new states you add are covered by your view logic.

## Verification

To verify that the selection field was extended:
1. Restart the Odoo server.
2. Update the module (`-u module_name`).
3. Inspect the field dropdown in the browser console or using Odoo's debug view.

# Manifest XML File Load Order Issues (External ID not found)

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | views                                      |
| Odoo Versions | All                                        |
| Severity      | 🔴 Critical                                |
| Last Verified | 2026-05-30                                 |
| Author        | ENG/Gamal Mansour                          |

**Tags:** `manifest`, `xml`, `load order`, `parent menu`, `external id`

---

## Problem

When installing or upgrading an Odoo module, a `ValueError: External ID not found in the system` or a `ParseError` occurs. This happens when an XML element (e.g., a menuitem, action, or view) refers to an external ID defined in another XML file that hasn't been parsed yet.

```
ValueError: External ID not found in the system: module_name.parent_menu_id
```

## Root Cause

Odoo loads XML files in the exact sequence they are declared in the `data` list in `__manifest__.py`. If a file (e.g., `dashboard_views.xml`) defines a menuitem with `parent="menu_root"` before the file defining `menu_root` (e.g., `menu_views.xml`) is loaded, Odoo's registry won't find the parent's external ID, causing installation to fail.

## Solution ✅

Ensure that the files in the `data` list in `__manifest__.py` are ordered by their dependencies. Files defining parent menus, actions, or base categories must be loaded **before** child menus, custom action views, or wizard elements.

Example of correct ordering in `__manifest__.py`:
```python
    'data': [
        'security/security_groups.xml',
        'security/ir.model.access.csv',
        'data/boq_stage_data.xml',
        'views/menu_views.xml',        # Loads root menus & core parent menus first
        'views/dashboard_views.xml',   # Loads dashboard menu, which references parent menu
        'views/job_order_views.xml',
        # ...
    ],
```

## ⚠️ Pitfalls

- **Circular Dependencies**: Do not create menus in two different files that reference each other's IDs as parents.
- **Inherited Views**: When inheriting views from other modules, make sure the other module is explicitly declared in `'depends'` in `__manifest__.py`.

## Verification

To verify, reinstall the module or run Odoo with:
```bash
odoo-bin -c <config> -u <module_name> -d <database>
```
The module should load successfully without any registry lookup errors.

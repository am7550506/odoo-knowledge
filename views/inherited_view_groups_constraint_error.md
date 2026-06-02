# Inherited View Cannot Have Groups Constraint Error

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | views                                      |
| Odoo Versions | 18, 19                                     |
| Severity      | 🔴 Critical                                 |
| Last Verified | 2026-06-02                                 |
| Author        | ENG/Gamal Mansour                          |

**Tags:** `views`, `groups`, `constraint`, `xml`, `inherit`, `upgrade`, `error`

---

## Problem

> When upgrading a module (e.g., `calendar_sms`) in Odoo 18 or 19, the server throws an RPC_ERROR parsing an inherited view XML file.

```
odoo.tools.convert.ParseError: while parsing .../addons/calendar_sms/views/calendar_views.xml:19
Inherited view cannot have 'Groups' define on the record. Use 'groups' attributes inside the view definition
```

## Root Cause

> Starting in Odoo 18, `ir.ui.view` enforces a strict constraint: an inherited view (`mode="extension"`) **cannot** have the `groups_id` field assigned to the view record itself. It MUST apply the groups via the `groups="group_xml_id"` attribute on elements inside the `arch` XML body.
>
> If this view already existed in the database (e.g. from a previous Odoo version) and a group was manually assigned to it from the UI (Settings -> Technical -> Views), or by another custom module modifying the record directly, updating the module will fail because the ORM `_check_groups` constraint detects the `groups_id` relation in the database while parsing the XML update.

## Solution ✅

> Remove the offending group assignment directly from the database for the specific view causing the issue.

```bash
# Step 1: Identify the view ID from the database using the module and view name from the error
psql -U gamal -d YOUR_DB_NAME -c "SELECT v.id, v.name, r.group_id FROM ir_ui_view v JOIN ir_ui_view_group_rel r ON r.view_id = v.id JOIN ir_model_data m ON m.res_id = v.id AND m.model='ir.ui.view' WHERE m.module='calendar_sms' AND m.name='view_calendar_event_tree_inherited';"

# Step 2: Delete the group association from the relational table for that view ID
psql -U gamal -d YOUR_DB_NAME -c "DELETE FROM ir_ui_view_group_rel WHERE view_id = YOUR_VIEW_ID;"
```

## ⚠️ Pitfalls

- **Do NOT try to fix the core XML:** The error is NOT in the source code of the module being upgraded (like `calendar_sms`), it is in the database (`ir_ui_view_group_rel`).
- **Never assign groups to inherited views in UI:** In Odoo 18+, assigning a group to an inherited view from the Technical menu will instantly trigger this constraint or cause future upgrades to fail.

## Verification

> Start Odoo and run the module upgrade again. The error should no longer appear.

```bash
# Verification command (example)
./odoo-bin -c odoo.conf -d YOUR_DB_NAME -u calendar_sms
```

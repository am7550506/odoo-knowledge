# Field Widgets Declaration in Kanban Views (KeyNotFoundError: Cannot find priority in this registry)

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | views                                      |
| Odoo Versions | 16, 17, 18, 19                             |
| Severity      | 🔴 Critical                                |
| Last Verified | 2026-05-30                                 |
| Author        | ENG/Gamal Mansour                          |

**Tags:** `views`, `kanban`, `widget`, `field`, `priority`, `kanban_state`

---

## Problem

When loading a Kanban view in Odoo, the client crashes with an OWL lifecycle error:
```
Caused by: KeyNotFoundError: Cannot find priority in this registry!
    at Registry.get (web.assets_web.min.js:3440:76)
    at Widget.parseWidgetNode (web.assets_web.min.js:10265:120)
```

## Root Cause

This error occurs when a standard field widget (such as `priority` or `state_selection`/`kanban_state`) is declared in a view using the `<widget>` tag:
```xml
<!-- INCORRECT -->
<widget name="priority" field="priority"/>
<widget name="kanban_state" field="kanban_state"/>
```
In Odoo 17+, the `<widget>` element is reserved for widgets registered in the JS `widget` registry, not the `fields` registry. Standard fields with widgets must be declared using the `<field>` tag with the `widget` attribute.

## Solution ✅

Replace the `<widget>` tags with `<field>` tags using the `widget` attribute:

```xml
<!-- CORRECT -->
<field name="priority" widget="priority"/>
<field name="kanban_state" widget="state_selection"/>
```

## ⚠️ Pitfalls

- **`kanban_state` vs `state_selection`**: When converting a Kanban state widget, the correct widget name in Odoo 17+ is `state_selection` (or `project_task_state_selection` for tasks), not `kanban_state`.
- **Options and Classes**: Ensure any styling classes or parameters are transferred to the `<field>` tag correctly.

## Verification

To verify:
1. Reload the Kanban view.
2. The cards should render correctly with the star priority indicator and status bullet, without throwing JS errors.

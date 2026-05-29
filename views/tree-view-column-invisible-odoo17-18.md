# 🎨 Migrating invisible to column_invisible in Tree/List Views (Odoo 17 & 18)

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | views                                      |
| Odoo Versions | 17, 18, 19                                 |
| Severity      | 🟡 Medium                                  |
| Last Verified | 2026-05-20                                 |
| Author        | ENG/Mohamed Saber                          |

**Tags:** `views`, `tree`, `list`, `invisible`, `column_invisible`, `xml`

---

## Problem

In Odoo 17 and 18, using the `invisible="..."` attribute directly on `<field>` elements within a `<tree>` or `<list>` view does not hide the entire column. Instead, it hides only the cell value, leaving empty slots in the table layout. In some cases, it can trigger UI glitches or layout misalignment.

## Root Cause

Odoo 17 introduced a clean separation of layout visibility:
- **`invisible`**: Used on fields within Form views (hides the field widget) or on buttons/groups inside List/Form views.
- **`column_invisible`**: A dedicated attribute introduced for `<field>` elements in `<tree>` or `<list>` views to hide the entire column dynamically or statically.

## Solution ✅

Replace the `invisible` attribute with `column_invisible` for `<field>` elements defined inside `<tree>` or `<list>` tags.

### Before

```xml
<tree>
    <field name="some_id" invisible="1"/>
    <field name="display_name"/>
</tree>
```

### After

```xml
<tree>
    <field name="some_id" column_invisible="1"/>
    <field name="display_name"/>
</tree>
```

> **Note:** If dynamically showing/hiding a column based on a parent/context field, use the `parent` prefix:
> `column_invisible="parent.some_parent_field == 'value'"`

## ⚠️ Pitfalls

- **Do NOT rename invisible on buttons:** Buttons (`<button>`) inside list/tree views STILL use `invisible` to hide individual row buttons. Applying `column_invisible` to buttons will not work.
- **Do NOT apply in Form/Search Views:** Only replace the attribute for `<field>` elements inside a `<tree>` or `<list>` context.

## Verification

Open the list/tree view in Odoo 17 or 18 dev mode, inspect the columns, and ensure columns marked as `column_invisible` are completely hidden from the table headers and body, without leaving blank slots.

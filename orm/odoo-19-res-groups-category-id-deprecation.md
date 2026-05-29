# Odoo 19 res.groups category_id Deprecation

## Problem 🔴
When creating a custom security group (`res.groups`) in Odoo 19 and assigning an `ir.module.category` using the `category_id` field in XML, an installation error occurs:
`ValueError: Invalid field 'category_id' in 'res.groups'`

## Cause ⚠️
Odoo 19 fundamentally changed how groups are categorized. The `category_id` field was completely removed from the `res.groups` model. Instead, Odoo 19 introduces a new model `res.groups.privilege`. Groups are now linked to categories via the `privilege_id` field.

## Solution ✅
For standard custom addons, creating a full privilege hierarchy is often unnecessary.
Simply **remove** the `category_id` field from your `<record model="res.groups">` definitions.
The group will still be created successfully and function normally, appearing under the "Other" or default categories in the UI.

**Incorrect (Odoo 18 and below):**
```xml
<record id="group_custom" model="res.groups">
    <field name="name">Custom Group</field>
    <field name="category_id" ref="module_category_custom"/>
</record>
```

**Correct (Odoo 19):**
```xml
<record id="group_custom" model="res.groups">
    <field name="name">Custom Group</field>
</record>
```

## Odoo Version
Odoo 19+

## Last Verified
2026-05-23

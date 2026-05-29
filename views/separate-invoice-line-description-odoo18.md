# Reverting Merged Invoice Line Description Widget in Odoo 18

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | views                                      |
| Odoo Versions | 18, 19                                     |
| Severity      | 🟡 Medium                                  |
| Last Verified | 2026-05-20                                 |
| Author        | ENG/Mohamed Saber                          |

**Tags:** `views`, `invoice`, `widget`, `one2many`, `product_label_section_and_note_field`

---

## Problem

In Odoo 18, the product description (`label` / `name`) in the invoice lines (tree view inside `account.move` form view) is no longer a separate column by default. Instead, it is merged and displayed directly below the product name inside the product column. 

This behavior is caused by a custom JS list renderer (`ProductLabelSectionAndNoteListRender`) and a product field widget (`product_label_section_and_note_field`) which filters out the `name` column from the active columns list and injects it as a nested text area under the product.

For many businesses, this is undesirable because they prefer the traditional grid layout where Description is an independent, separate column next to the Product column.

## Root Cause

1. The `invoice_line_ids` field is configured with the `product_label_section_and_note_field_o2m` widget.
2. The `product_id` field inside the invoice lines list is configured with the `product_label_section_and_note_field` widget.
3. The custom `ProductLabelSectionAndNoteListRender` list renderer dynamically removes the `name` column from the list if the product column is active.

## Solution ✅

Create a separate addon (e.g., `imw_separate_invoice_description`) that inherits the `account.view_move_form` view and reverts both widgets to Odoo's standard, separate layout components:

1. Revert `invoice_line_ids` widget to Odoo's standard `section_and_note_one2many` widget.
2. Revert the `product_id` column's widget to standard `many2one`.

### Inherited View XML

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <record id="view_move_form_inherit_separate_description" model="ir.ui.view">
        <field name="name">account.move.form.inherit.separate.description</field>
        <field name="model">account.move</field>
        <field name="inherit_id" ref="account.view_move_form"/>
        <field name="arch" type="xml">
            <!-- 1. Revert invoice line O2M widget to standard sections/notes renderer -->
            <xpath expr="//field[@name='invoice_line_ids']" position="attributes">
                <attribute name="widget">section_and_note_one2many</attribute>
            </xpath>

            <!-- 2. Revert product_id widget to standard many2one -->
            <xpath expr="//field[@name='invoice_line_ids']/list//field[@name='product_id']" position="attributes">
                <attribute name="widget">many2one</attribute>
            </xpath>
        </field>
    </record>
</odoo>
```

This restores the `name` column as a fully functional and separate text column next to the product while preserving sections and notes functionality.

## ⚠️ Pitfalls

- **XPath compatibility:** Ensure you target the `<list>` node (and not `<tree>`), as Odoo 18 forms explicitly define the nested grid view inside `invoice_line_ids` using a `<list>` element.
- **Dependency:** Ensure the module inherits from `account` and includes `'account'` in `'depends'` list in `__manifest__.py`.

## Verification

1. Install the module.
2. Navigate to **Invoicing / Accounting** -> **Customer Invoices** -> Create/Edit invoice.
3. Confirm that the Description field is now displayed as a separate column next to the Product column, rather than nested under it.

## References

- Related view: `addons/account/views/account_move_views.xml`
- Custom list renderer: `addons/account/static/src/components/product_label_section_and_note_field/product_label_section_and_note_field.js`

# QWeb Report Empty Invoice Lines due to display_type Changes in Odoo 17 & 18

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | views                                      |
| Odoo Versions | 17, 18, 19                                 |
| Severity      | 🔴 Critical                                |
| Last Verified | 2026-05-20                                 |
| Author        | ENG/Mohamed Saber                          |

**Tags:** `qweb`, `report`, `invoice`, `lines`, `display_type`, `empty-lines`

---

## Problem

After upgrading or migrating a custom invoice print report to Odoo 17 or Odoo 18, the invoice lines table renders completely empty or misses the regular product lines, even though the total and other invoice details load perfectly.

## Root Cause

In Odoo 16 and below, regular (accountable) invoice lines had `display_type` set to `False` or empty. Custom QWeb report templates checked for regular lines using:
```xml
<t t-if="not line.display_type">
```
Starting from Odoo 17, Odoo introduced a dedicated selection value for regular accountable lines, setting the default `display_type` to `'product'`:
```python
display_type = fields.Selection(
    selection=[
        ('product', 'Product'),
        ('line_section', 'Section'),
        ('line_note', 'Note'),
        ...
    ],
    default='product',
)
```
Consequently, because `line.display_type` is `'product'`, the condition `not line.display_type` evaluates to `False`. This causes the QWeb engine to skip rendering the accountable product lines completely.

## Solution ✅

Modify the QWeb template inside the `<t t-foreach="lines" t-as="line">` loop to support both empty values (for legacy/draft lines) and the `'product'` selection value.

### Before (Odoo 16 & below)
```xml
<t t-if="not line.display_type" name="account_invoice_line_accountable">
```

### After (Odoo 17, 18 & 19 compatible)
```xml
<t t-if="not line.display_type or line.display_type == 'product'" name="account_invoice_line_accountable">
```
*Note: Using `not line.display_type or line.display_type == 'product'` ensures perfect backward compatibility and bulletproof execution.*

## ⚠️ Pitfalls

- **Do not use `line.display_type == 'product'` alone** if there is any chance that some migrated lines or custom lines have an empty `display_type` (which evaluates to False). The combined condition is always safer.
- Double-check for typos in cell alignment classes, such as `text-lift` instead of `text-left`.

## Verification

1. Regenerate the PDF invoice print-out.
2. Verify that all product lines, description, quantities, and prices now render correctly in the lines table.

# 📄 account.move _get_reconciled_info_JSON_values Migration to Odoo 18

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | upgrade                                    |
| Odoo Versions | 17, 18, 19                                 |
| Severity      | 🔴 Critical                                |
| Last Verified | 2026-05-20                                 |
| Author        | ENG/Mohamed Saber                          |

**Tags:** `upgrade`, `migration`, `account.move`, `qweb`, `report`, `reconciled-payments`

---

## Problem

When trying to print or render custom invoice/vendor bill PDF reports that display details of reconciled payments/reconciled entries, you will encounter the following error on Odoo 18:

```
AttributeError: 'account.move' object has no attribute '_get_reconciled_info_JSON_values'
```

## Root Cause

In older Odoo versions (pre-v17), custom reports retrieved the payment list associated with an invoice by calling the helper method `_get_reconciled_info_JSON_values()` on the `account.move` model. In Odoo 17/18, this method has been completely removed. Reconciled payment details are now exposed directly through the standard JSON field `invoice_payments_widget` on the `account.move` model.

## Solution ✅

Replace the deprecated method call with a lookup on the `invoice_payments_widget` field, specifically extracting the list from its `content` dictionary key.

### Before

```xml
<t t-set="payments_vals" t-value="o.sudo()._get_reconciled_info_JSON_values()"/>
```

### After

```xml
<t t-set="payments_vals" t-value="o.sudo().invoice_payments_widget and o.sudo().invoice_payments_widget['content'] or []"/>
```

## ⚠️ Pitfalls

- **Ensure fallback fallback:** Always use `and ... or []` as a fallback. For draft or unpaid invoices, `invoice_payments_widget` will be empty/false, and trying to access its dictionary keys directly will cause a rendering exception.
- **Maintain sudo():** Keep the `sudo()` context to allow portal users or non-accounting users to download and view their paid invoices without triggering `AccessError` when fetching the payment widget content.

## Verification

Re-render the customized invoice or vendor bill PDF reports for fully paid or partially paid moves and verify that the payment date and amounts are displayed properly without any exceptions.

## References

- Odoo core standard view: `addons/account/views/report_invoice.xml`

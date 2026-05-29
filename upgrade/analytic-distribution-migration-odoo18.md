# 🚀 account.move.line analytic_account_id Migration to Odoo 18

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | upgrade                                    |
| Odoo Versions | 18, 19                                     |
| Severity      | 🔴 Critical                                |
| Last Verified | 2026-05-20                                 |
| Author        | ENG/Mohamed Saber                          |

**Tags:** `upgrade`, `migration`, `account.move.line`, `analytic_distribution`

---

## Problem

When writing custom modules that create or update billing/invoice lines (`account.move.line`) and pass `analytic_account_id` directly, you will encounter the following error on Odoo 18:

```
ValueError: Invalid field 'analytic_account_id' on model 'account.move.line'
```

## Root Cause

In Odoo 18 (and starting from Odoo 17/18), the traditional `analytic_account_id` field on the `account.move.line` model has been completely removed. It is replaced by the JSON field `analytic_distribution`, which allows splitting costs and revenues between multiple analytic accounts.

## Solution ✅

When creating or updating `account.move.line` records, instead of passing `analytic_account_id` as a Many2one ID, you must pass the dictionary values to `analytic_distribution`. The keys of the dictionary must be string representation of the analytic account IDs, and the values must be floats or integers representing the distribution percentage (usually 100 for a direct/sole account assignment).

### Before (Odoo 16/15 and older)

```python
invoice.invoice_line_ids = [
    (0, 0, {
        "product_id": line.products_id.id,
        "quantity": line.quantity,
        "analytic_account_id": rec.analytic_account_id.id,
    })
]
```

### After (Odoo 18)

```python
invoice.invoice_line_ids = [
    (0, 0, {
        "product_id": line.products_id.id,
        "quantity": line.quantity,
        "analytic_distribution": {str(rec.analytic_account_id.id): 100} if rec.analytic_account_id else False,
    })
]
```

## ⚠️ Pitfalls

- **JSON Keys must be Strings:** Passing integers as keys in the `analytic_distribution` dictionary (e.g., `{rec.analytic_account_id.id: 100}`) might lead to serialization issues or unexpected behavior. Always cast the ID to string using `str(id)`.
- **Handling False/Empty values:** If the source analytic account is empty, ensure you pass `False` or do not include the key to avoid writing empty strings or invalid dictionary values.
- **Many2one Recordset Adaptation Error (Odoo 18 / psycopg2):** When copying or mapping other fields (like `uom_id` or `otherUnitMeasure`) during invoice line creation dictionary writes, ensure you always pass the integer ID (`.id`) rather than passing the recordset itself. Passing raw recordset objects (e.g., `line.other_uom_id`) inside the `[(0, 0, vals)]` dictionary will raise a database type adaptation error: `psycopg2.ProgrammingError: can't adapt type 'uom.uom'`.

## Verification

Run Odoo, trigger the action that creates the invoice (e.g., Daily Log Report invoice generation), and verify that the invoice is successfully created with correct analytic account distribution details on the invoice lines.

## References

- [Odoo 18 Documentation on Analytic Accounts](https://www.odoo.com/)

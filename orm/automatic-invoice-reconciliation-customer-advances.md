# Automatic Invoice Reconciliation with Customer Advances

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | orm                                        |
| Odoo Versions | 17, 18, 19                                 |
| Severity      | 🟡 Medium                                  |
| Last Verified | 2026-05-20                                 |
| Author        | ENG/Mohamed Saber                          |

**Tags:** `orm`, `account`, `reconciliation`, `invoice`, `customer-advance`

---

## Problem

In custom e-commerce or Shopify Odoo integrations, customer payments are often recorded as advance payments in a liability or gateway-specific account. When the invoice is subsequently created (either manually or automatically during delivery validation), the invoice remains in "Open" (unpaid) state, requiring manual accounting intervention to match and reconcile it with the advance payment.

## Root Cause

1. Customer advances are typically credited to a different account (e.g. "Advances from Customers" liability account) than the default customer receivable account (`property_account_receivable_id`).
2. Odoo requires lines to be on the exact same account in order to reconcile them.
3. Simply posting the transfer entry that moves the advance to the receivable account does not trigger Odoo's matching engine; the two posted lines must be explicitly linked via Odoo's `.reconcile()` method.

## Solution ✅

Implement a smart automatic reconciliation logic inside the invoice creation or posting hooks (`_create_invoices` or `action_post` overrides):

1. **Verify Account Match:**
   - Fetch the credit line of the Advance Payment entry.
   - Fetch the receivable line of the generated Invoice.
   - Compare their accounts.

2. **Reconcile Directly or via Transfer Entry:**
   - **Case A: Accounts Match:** Reconcile the credit line of the advance payment entry directly with the invoice's receivable line.
   - **Case B: Accounts Differ:** Create an intermediate Transfer/Reconciliation Entry to move the balance, and then reconcile the invoice with the transfer entry.

```python
# 1. Find the credit line of the Advance Payment entry
credit_line = advance_entry.line_ids.sudo().filtered(
    lambda l: l.credit > 0 and not l.reconciled
)[:1]

# 2. Find the receivable line in the invoice
invoice_line = invoice.line_ids.sudo().filtered(
    lambda l: l.account_id.account_type == "asset_receivable" and not l.reconciled
)[:1]

if credit_line and invoice_line:
    if credit_line.account_id == invoice_line.account_id:
        # SAME ACCOUNT: Reconcile directly! No intermediate entry needed.
        (invoice_line + credit_line).sudo().reconcile()
    else:
        # DIFFERENT ACCOUNTS: Create the Reconcile/Transfer entry
        entry = self.env["account.move"].sudo().create({
            "move_type": "entry",
            "sale_id": rec.id,
            "ref": "Reconcile Payment For " + str(rec.name),
            "invoice_date": fields.date.today(),
            "journal_id": sale_journal.id,
            "line_ids": [
                (0, 0, {
                    "account_id": credit_line.account_id.id,
                    "partner_id": rec.partner_id.id,
                    "debit": invoice_amount,
                }),
                (0, 0, {
                    "account_id": invoice_line.account_id.id,
                    "partner_id": rec.partner_id.id,
                    "credit": invoice_amount,
                }),
            ],
        })
        entry.action_post()
        
        # Reconcile the invoice line with the transfer entry line
        reconcile_line = entry.line_ids.sudo().filtered(
            lambda l: l.account_id.account_type == "asset_receivable" and not l.reconciled
        )
        if invoice_line and reconcile_line:
            (invoice_line + reconcile_line).sudo().reconcile()
```

## ⚠️ Pitfalls

- **Access Rights:** Non-accounting users (like warehouse operators validating pickings) will trigger this method if the invoice is created on delivery. Wrap all line queries and the `.reconcile()` call in `sudo()` to bypass accounting permissions.
- **Duplicate Checks:** If invoices are re-created or edited, ensure the transfer entry check is performed to prevent duplicate accounting entries.
- **Rounding and Partial Amounts:** Ensure the transfer entry uses the actual posted invoice's total amount (`invoice.amount_total`), not the estimated order amount, to prevent leaving partial open amounts.

## Verification

Create a Sales Order with a Shopify gateway payment configured, validate the delivery (or click Create Invoice), and confirm that:
1. The invoice is automatically posted.
2. The transfer entry is created.
3. The invoice payment status changes to "Paid" (reconciled).

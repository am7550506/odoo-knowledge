# Forwarding Carrier and Banking Info in Refund Request & Credit Note

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | orm                                        |
| Odoo Versions | 17, 18, 19                                 |
| Severity      | 🟡 Medium                                  |
| Last Verified | 2026-05-20                                 |
| Author        | ENG/Mohamed Saber                          |

**Tags:** `orm`, `refund`, `credit-note`, `carrier`, `banking`

---

## Problem

When using a customized refund request flow in Odoo (where a `refund.request` model is created from a Sales Order or a Picking, and subsequently generates a Credit Note `account.move`), the carrier and banking details (e.g. payment type, bank, IBAN, account name) are often lost or not propagated correctly unless explicitly handled at all creation entry points.
Additionally, when a new payment type selection option is introduced (e.g. "Other"), banking and wallet fields must be dynamically hidden/shown, a description field must be displayed, and this description must propagate all the way from the return wizard through stock pickings to the credit note.

## Root Cause

Odoo does not automatically map customized fields (like custom banking fields on Sale Order/Refund Request/Credit Note) or standard carrier fields across these different models. Manual form creation from action buttons loses context values unless default keys are explicitly supplied in the action context. Automated creation via code needs explicit mappings in the `create` vals dictionary.

## Solution ✅

Ensure carrier and banking fields are propagated across the workflow:

1. **Extend Selection and Add Description Field:** Add the selection options (e.g. `other`) and `payment_type_desc` (e.g., `fields.Text`) to all relevant models (wizard, sale.order, stock.picking, refund.request, account.move).

2. **Context Defaults in Actions:** When opening the Refund Request action view from the Sale Order, specify the defaults in the context:
    ```python
    "default_carrier_id": rec.carrier_id.id or rec.carrier_account_id.id or False,
    "default_payment_type": rec.payment_type or False,
    "default_payment_type_desc": rec.payment_type_desc or False,
    "default_bank_id": rec.bank_id.id if rec.bank_id else False,
    "default_bank_account_name": rec.bank_account_name or False,
    "default_iban": rec.iban or False,
    ```

3. **Onchange mapping:** Add `@api.onchange('sale_id')` to `refund.request` to dynamically retrieve defaults if a user changes or selects the Sale Order manually in the form view:
    ```python
    @api.onchange('sale_id')
    def _onchange_sale_id(self) -> None:
        for rec in self:
            if rec.sale_id:
                rec.carrier_id = rec.sale_id.carrier_id.id or rec.sale_id.carrier_account_id.id or False
                rec.payment_type = rec.sale_id.payment_type or False
                rec.payment_type_desc = rec.sale_id.payment_type_desc or False
                rec.bank_id = rec.sale_id.bank_id.id if rec.sale_id.bank_id else False
                rec.bank_account_name = rec.sale_id.bank_account_name or False
                rec.iban = rec.sale_id.iban or False
    ```

4. **Creation propagation (Credit Note):** In the credit note generation logic, make sure to map the banking fields and the description to the custom fields on the `account.move` model:
    ```python
    credit_note = self.env["account.move"].create({
        "move_type": "out_refund",
        "carrier_account_id": self.carrier_id.id or False,
        "payment_type_account": self.payment_type or False,
        "payment_type_desc": self.payment_type_desc or False,
        "bank_id": self.bank_id.id or False,
        "bank_account_name": self.bank_account_name or False,
        "iban": self.iban or False,
    })
    ```

5. **Dynamic XML Field Visibility and Requirements:** Use Odoo 17+ dynamic attributes to handle field visibility and requirement constraints based on selection values (e.g. `payment_type != 'bank'`):
    ```xml
    <field name="payment_type"/>
    <field name="payment_type_desc" invisible="payment_type != 'other'" required="payment_type == 'other'"/>
    <field name="bank_id" invisible="payment_type != 'bank'" required="payment_type == 'bank'"/>
    ```

## ⚠️ Pitfalls

- **Conflicting selection names:** The selection field names might differ between models (e.g., `payment_type` on `refund.request` vs `payment_type_account` on `account.move`). Make sure to map them correctly.
- **Multiple carrier fields:** Check if the carrier field name is `carrier_id` or `carrier_account_id` in target models.
- **Variable scoping in automated functions:** When copying values from picking/sale order to credit note/refund request, ensure that you refer to the correctly scoped record instances (e.g., `self.sale_id` or `self`) instead of undefined variables.

## Verification

Verify that manually creating a Refund Request from a Sale Order or automatically from picking propagates all details to the Refund Request, and then confirming the request propagates the correct details to the resulting Credit Note form.


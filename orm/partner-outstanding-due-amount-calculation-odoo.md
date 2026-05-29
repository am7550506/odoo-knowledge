# 💰 Partner Outstanding Due Amount Calculation & Stat Button

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | orm                                        |
| Odoo Versions | All                                        |
| Severity      | 🟢 Low                                     |
| Last Verified | 2026-05-20                                 |
| Author        | ENG/Mohamed Saber                          |

**Tags:** `orm`, `partner`, `due-amount`, `credit`, `debit`, `views`, `stat-button`

---

## Problem

In Odoo, there is no single out-of-the-box field on `res.partner` that represents the net due amount (Receivables minus Payables) visible outside of Enterprise followup models. Customers often request this field to be visible in the partner form view, next to credit limit fields or inside header stat buttons with direct navigation to unpaid invoices.

## Solution ✅

Implement a custom computed field `due_amount` inside a lightweight custom module and expose it as a header stat button.

### 1. Python Model Definition

Calculate `due_amount` dynamically based on the pre-computed `credit` and `debit` fields to keep it fast, and define an action to drill down to unpaid invoices:

```python
# -*- coding: utf-8 -*-
from odoo import models, fields, api

class ResPartner(models.Model):
    _inherit = 'res.partner'

    due_amount = fields.Monetary(
        string='Due Amount',
        compute='_compute_due_amount',
        groups='account.group_account_invoice,account.group_account_readonly',
        help='Total outstanding due amount (Receivables minus Payables).'
    )

    @api.depends('credit', 'debit')
    def _compute_due_amount(self) -> None:
        for partner in self:
            partner.due_amount = partner.credit - partner.debit

    def action_view_due_invoices(self) -> dict:
        self.ensure_one()
        action = self.env["ir.actions.actions"]._for_xml_id("account.action_move_out_invoice_type")
        action['domain'] = [
            ('partner_id', 'child_of', self.id),
            ('state', '=', 'posted'),
            ('payment_state', 'in', ('not_paid', 'partial')),
            ('move_type', 'in', ('out_invoice', 'out_refund'))
        ]
        action['context'] = {
            'default_move_type': 'out_invoice',
            'move_type': 'out_invoice',
            'journal_type': 'sale',
        }
        return action
```

### 2. XML View Definition

Display the field in the invoicing sheet and inside the top header stat buttons:

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <record id="view_partner_form_inherit_due_amount_btn" model="ir.ui.view">
        <field name="name">res.partner.form.inherit.due.amount.btn</field>
        <field name="model">res.partner</field>
        <field name="inherit_id" ref="base.view_partner_form"/>
        <field name="priority" eval="10"/>
        <field name="arch" type="xml">
            <div name="button_box" position="inside">
                <button type="object" class="oe_stat_button" icon="fa-money" name="action_view_due_invoices"
                    groups="account.group_account_invoice,account.group_account_readonly"
                    invisible="due_amount == 0">
                    <div class="o_form_field o_stat_info">
                        <span class="o_stat_value">
                            <field name="due_amount" widget="monetary" options="{'currency_field': 'currency_id'}"/>
                        </span>
                        <span class="o_stat_text">Due Amount</span>
                    </div>
                </button>
            </div>
        </field>
    </record>
</odoo>
```

---

## ⚠️ Pitfalls

- **Avoid Heavy SQL Queries:** Never perform raw loops or searches on `account.move` to calculate the balance inside `_compute_due_amount` if you can leverage the pre-indexed `credit` and `debit` fields. That will degrade performance drastically on databases with massive invoice history.
- **Respect Multi-Currency:** Always add `widget="monetary"` and pass `options="{'currency_field': 'currency_id'}"` to format values accurately with regional currency symbols.

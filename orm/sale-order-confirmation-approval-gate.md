# Blocking Sale Order Confirmation on Custom Approval Conditions

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | orm                                        |
| Odoo Versions | 17, 18, 19                                 |
| Severity      | 🟢 Low                                     |
| Last Verified | 2026-09-05                                 |
| Author        | ENG/Gamal Mansour                          |

**Tags:** `sale`, `sale.order`, `approval`, `action_confirm`, `workflow`, `hook`, `groups`

---

## Problem

> Client wants extra approval gate(s) before a Sale Order can move from
> Quotation to Sales Order (e.g. "Sales Manager approval" and "Accounts
> Manager approval", each restricted to its own security group) — without
> touching the actual `draft`/`sent`/`sale`/`cancel` state machine.

## Root Cause

> `sale.order.action_confirm()` already delegates the "can this be
> confirmed at all" check to a small hook:
> `_confirmation_error_message()` (returns `False` if OK, or an error
> string). Odoo's own `sale` module uses it for its one built-in check
> (order line missing a product). Overriding `action_confirm()` itself
> is unnecessary and riskier (blast radius on a core transactional
> model) — the hook already exists precisely for this.

## Solution ✅

### 1. Model — extend the hook, not `action_confirm`

```python
class SaleOrder(models.Model):
    _inherit = "sale.order"

    sales_manager_approved = fields.Boolean(copy=False, tracking=True)
    accounts_manager_approved = fields.Boolean(copy=False, tracking=True)

    def action_sales_manager_approve(self):
        if not self.env.user.has_group("my_module.group_x"):
            raise AccessError(_("Not allowed."))
        self.write({"sales_manager_approved": True})

    def _confirmation_error_message(self):
        error_msg = super()._confirmation_error_message()
        if error_msg:
            return error_msg
        if not self.sales_manager_approved:
            return _("This quotation still needs Sales Manager approval.")
        return False
```

`_confirmation_error_message()` is called `ensure_one()`-style, once per
order, from inside `action_confirm()`'s loop — so `self` is always a
single record here.

### 2. View — buttons anchored on the existing Confirm button's id

The header has **two** `action_confirm` buttons in the arch (one
`id="action_confirm"` shown when `state == 'sent'`, one un-id'd shown
when `state == 'draft'`). `locate_node()` for an `xpath` spec only ever
returns the **first** match (`nodes[0]`), so pick the id'd one as the
anchor and rely on document order — a single `position="before"` insert
there lands before *both* variants in the DOM, so it works regardless
of which one is currently visible:

```xml
<xpath expr="//button[@id='action_confirm']" position="before">
    <button name="action_sales_manager_approve" type="object"
            string="Sales Manager Approval" class="btn-secondary"
            groups="my_module.group_x"
            invisible="state not in ('draft','sent') or sales_manager_approved"/>
</xpath>
```

No need to duplicate the button once per Confirm variant.

## ⚠️ Pitfalls

- **`groups=` on the button is not enough security** — it only hides
  the button in the view. Also check `self.env.user.has_group(...)` in
  the button's own method and raise `AccessError` if not a member,
  otherwise the action is callable via RPC by anyone.
- **Editing the order after approval silently keeps the stale
  approval.** If order lines / partner / pricelist can change after an
  approval was granted but before Confirm is clicked, override
  `write()` to reset the approval booleans when those fields are in
  `vals` — otherwise a manager's approval on order content X can be
  used to wave through content Y.
- Only override `_confirmation_error_message()` — do **not** override
  `action_confirm()` wholesale; that duplicates logic (email sending,
  `_prepare_confirmation_values()`, lock handling) that the hook
  approach gets for free via `super()`.
- This only gates the **Confirm** action. `action_quotation_send` (the
  "Send" button) is untouched — if "before confirmation" is meant to
  also block sending the quotation, that needs a separate check.

## Verification

```bash
./odoo-bin -c <conf> -d <db> -u <module> --test-enable --test-tags /<module> --stop-after-init
```
Manually: create a quotation, click Confirm with no approval → blocked
with the custom message; approve as each required group; Confirm now
succeeds and the order becomes a Sales Order.

## References

- Odoo core: `addons/sale/models/sale_order.py::action_confirm`,
  `_confirmation_error_message`
- Related file: `orm/state-draft-confirm-pattern.md`
- Related file: `orm/write-override-atomicity-pattern.md`

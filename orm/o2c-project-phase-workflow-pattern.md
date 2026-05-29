# O2C Lifecycle Module – Project Phase Workflow Pattern

| Field | Value |
|-------|-------|
| **Category** | ORM |
| **Severity** | 🟡 Medium |
| **Odoo Versions** | 19 |
| **Tags** | `orm`, `project`, `workflow`, `selection`, `statusbar`, `write-override` |
| **Author** | ENG/Mohamed Saber |
| **Created** | 2026-05-21 |
| **Last Verified** | 2026-05-21 |

---

## Problem 🔍

Implementing a multi-phase lifecycle workflow on `project.project` using a Selection field with a clickable statusbar, where phase transitions need business rule validation (document checks, payment checks, conditional branching).

## Solution ✅

### Pattern: Selection + Clickable Statusbar + Write Override

1. **Define a Selection field** with `group_expand` to show all phases in grouped views:
```python
o2c_phase = fields.Selection(
    selection=[...],
    group_expand='_group_expand_o2c_phase',
)
```

2. **Override `write()`** to intercept statusbar clicks and validate transitions:
```python
def write(self, vals):
    if 'o2c_phase' in vals:
        for project in self:
            project._validate_o2c_transition(vals['o2c_phase'])
    return super().write(vals)
```

3. **Centralize validation** in a single method that checks:
   - Allowed transition graph (current → target)
   - Per-phase business conditions

### Pattern: Milestone-to-Invoice Auto-Tracking

Use a stored computed field on `project.milestone` that depends on the linked `account.move.payment_state`:
```python
invoice_status = fields.Selection(
    compute='_compute_invoice_status', store=True,
)

@api.depends('invoice_id', 'invoice_id.payment_state', 'invoice_id.state')
def _compute_invoice_status(self):
    ...
```

### Pattern: Delivery Note → Milestone Automation

When a Delivery Note is approved:
1. Mark milestone `is_reached = True` via `sudo().write()`
2. Schedule activity on the **project** (not milestone, as milestone doesn't inherit `mail.activity.mixin`)

## ⚠️ Pitfalls

1. **`project.milestone` does NOT inherit `mail.activity.mixin`** — you cannot schedule activities directly on milestones. Use the project record instead.
2. **`account_id` already exists** on `project.project` as the analytic account — don't create a duplicate `cost_center_id`. Relabel in the view with `string="Cost Center"`.
3. **Many2many attachment fields** need unique relation table names to avoid conflicts: use module-prefixed names like `project_o2c_delivery_plan_att_rel`.
4. **Payment state values** in Odoo 19: `not_paid`, `in_payment`, `paid`, `partial`, `reversed`, `blocked`, `invoicing_legacy`.
5. **`res.groups` in Odoo 19: `category_id` → `privilege_id`** — Groups no longer use `ir.module.category` directly. Create a `res.groups.privilege` record and use `privilege_id` instead of `category_id`.
6. **Search view `<group>` in Odoo 19 has NO attributes** — do NOT use `string="..."` or `expand="0"` on `<group>` inside `<search>`. Just use bare `<group>`.
7. **`res.groups.users` → `res.groups.user_ids`** — In Odoo 19, the field is `user_ids` (not `users`).

## References

- Module: `zakham/project_o2c`
- Technical Spec: `O2C_Technical_Spec.md`

# State Machine: Draft/Confirm Workflow Pattern in Odoo

| Field         | Value                                      |
|---------------|--------------------------------------------| 
| Category      | orm                                        |
| Odoo Versions | 16, 17, 18, 19                             |
| Severity      | 🟡 Medium                                  |
| Last Verified | 2026-06-02                                 |
| Author        | ENG/Ahmed Mohamed                          |

**Tags:** `orm`, `state`, `draft`, `confirm`, `workflow`, `readonly`, `tracking`, `chatter`

---

## Problem

> A custom model needs a simple approval workflow where:
> 1. All field changes are logged in the Chatter (audit trail).
> 2. A "Confirm" button locks all fields to prevent tampering.
> 3. A "Set to Draft" button allows re-editing.

## Root Cause

> Odoo provides `mail.thread` mixin for tracking. State-based `readonly` in views prevents edits without needing `@api.constrains`. The `state` field itself must be `readonly=True` on the field level so only Python methods can change it — not the user directly.

## Solution ✅

### 1. Model (`models/your_model.py`)

```python
from odoo import api, fields, models, _
from odoo.exceptions import UserError

class YourModel(models.Model):
    _name = "your.model"
    _inherit = ["mail.thread", "mail.activity.mixin"]

    # All fields that need audit trail must have tracking=True
    your_field = fields.Char(string="Your Field", tracking=True)

    state = fields.Selection(
        [("draft", "Draft"), ("confirmed", "Confirmed")],
        string="Status",
        required=True,
        readonly=True,   # IMPORTANT: readonly here so only Python can set it
        copy=False,
        tracking=True,
        default="draft",
    )

    def action_confirm(self) -> None:
        """Confirm the record and lock all fields."""
        for rec in self:
            if not rec.your_field:
                raise UserError(_("Cannot confirm without required data."))
            rec.state = "confirmed"

    def action_draft(self) -> None:
        """Reset record back to Draft."""
        self.write({"state": "draft"})
```

### 2. XML View (`views/your_model_views.xml`)

```xml
<form>
    <header>
        <!-- Confirm button: only visible in draft -->
        <button name="action_confirm" type="object" string="Confirm"
                class="btn-primary" invisible="state != 'draft'"/>
        <!-- Draft button: only visible when confirmed -->
        <button name="action_draft" type="object" string="Set to Draft"
                invisible="state != 'confirmed'"/>
        <!-- Status bar always visible -->
        <field name="state" widget="statusbar" statusbar_visible="draft,confirmed"/>
    </header>
    <sheet>
        <group>
            <!-- Apply readonly based on state -->
            <field name="your_field" readonly="state != 'draft'"/>
        </group>
        <!-- For One2many lines, apply readonly on the field tag -->
        <field name="line_ids" readonly="state != 'draft'">
            <list editable="bottom">
                <field name="name"/>
            </list>
        </field>
    </sheet>
    <chatter/>
</form>
```

### 3. List View — Add badge for state

```xml
<list>
    <field name="name"/>
    <field name="state" widget="badge"
           decoration-info="state == 'draft'"
           decoration-success="state == 'confirmed'"/>
</list>
```

## ⚠️ Pitfalls

- **Don't forget `readonly=True` on the `state` field definition** in the model. Without this, a user could manually change the state via the UI or RPC without going through your action methods.
- **`tracking=True` requires `_inherit = ["mail.thread"]`** — if the mixin is missing, Odoo silently ignores tracking without raising an error, and you'll wonder why the Chatter is empty.
- **Use `write()` in `action_draft()`, not direct assignment** — `self.write({"state": "draft"})` ensures `mail.thread` tracking is triggered correctly on multi-record sets.
- **One2many lines readonly**: Apply `readonly="state != 'draft'"` on the `<field>` tag (One2many level), not inside the `<list>`. If applied inside the list, it affects column headers but not the add/delete row functionality.
- **`copy=False` on state** — always add `copy=False` to the state field to ensure duplicated records start as `draft`, not in their previous confirmed state.

## Verification

> After upgrading the module:
> 1. Create a new record → Confirm button should appear.
> 2. Make a change → Check Chatter for the tracking message.
> 3. Click Confirm → All fields become read-only, only "Set to Draft" is visible.
> 4. Click Set to Draft → Fields become editable again.

## References

- Odoo ORM docs: `mail.thread` mixin
- Related file: `orm/write-override-atomicity-pattern.md`

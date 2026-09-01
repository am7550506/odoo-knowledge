# Do NOT Use `_()` Inside Field Definitions (Class Body)

| Field         | Value                          |
|---------------|--------------------------------|
| Category      | orm                            |
| Odoo Versions | All (14, 15, 16, 17, 18, 19)  |
| Severity      | 🔴 Critical                    |
| Last Verified | 2026-09-01                     |
| Author        | ENG/Gamal Mansour              |

**Tags:** `translation`, `i18n`, `fields`, `_()`, `class-body`, `bug`, `_sql_constraints`, `models.Constraint`

---

## Problem

Using `_()` inside field `string=` or `help=` parameters causes translations to be evaluated **at module import time**, not at request time. This means the string is always in the server's default language, ignoring the user's language completely.

```python
# ❌ WRONG — evaluated once at import, always returns server language
class CustomerLevel(models.Model):
    name = fields.Char(string=_('Level Name'))        # BUG
    color = fields.Integer(string=_('Color Index'))   # BUG

    _sql_constraints = [
        ('unique_name', 'UNIQUE(name)', _('Name must be unique!'))  # BUG
    ]
```

No error is raised — it silently fails to translate for users with different languages.

## Root Cause

In Python, class bodies are executed once when the module is first imported. `_()` at that point resolves against the server's locale at startup, not the user's `lang` context at request time. Odoo's i18n system works differently for field labels: it reads `string=` as a raw English string and translates it on-the-fly using `.po` files.

## Solution ✅

Remove `_()` from `string=`, `help=`, and `_sql_constraints` messages entirely. Use plain English strings — Odoo's i18n handles the translation automatically via `.po` files:

```python
# ✅ CORRECT
class CustomerLevel(models.Model):
    name = fields.Char(string='Level Name')        # translated via i18n
    color = fields.Integer(string='Color Index')   # translated via i18n

    _sql_constraints = [
        ('unique_name', 'UNIQUE(name)', 'Name must be unique!')  # plain string
    ]
```

Keep `_()` only inside **method bodies** where it runs at request time:

```python
# ✅ Correct use of _() — inside a method
def action_confirm(self):
    raise UserError(_('Cannot confirm without lines.'))
```

## ⚠️ Pitfalls

- This bug is **silent** — no error, no warning. The string just never translates.
- It also affects `_sql_constraints` error messages and `Selection` labels if wrapped.
- Affects: `string=`, `help=`, `selection` list items in field defs, and `_sql_constraints`.
- **Do NOT** wrap `compute=`, `inverse=`, `search=` — those are method names, not strings.
- 🔴 **Odoo 19: `_sql_constraints` (the list-of-tuples form shown above) is no longer
  supported.** It loads without a hard error but logs
  `WARNING ... Model attribute '_sql_constraints' is no longer supported, please define
  model.Constraint on the model.` and **the constraint is never created in the database** —
  it silently does nothing, so duplicate/invalid rows go through uncaught. Confirmed by
  installing a module with the old syntax on 19.0: no unique index existed on the table
  afterwards. Replace with a `models.Constraint` class attribute instead:
  ```python
  # ❌ Dead on 19.0 — loads, warns, enforces nothing
  _sql_constraints = [
      ('unique_name', 'UNIQUE(name)', 'Name must be unique!'),
  ]

  # ✅ Odoo 19 — same rule, still just a plain string (no _())
  _unique_name = models.Constraint(
      'UNIQUE(name)',
      'Name must be unique!',
  )
  ```
  The translation rule in this entry (never wrap the message in `_()`) still applies
  identically to the new form. Attribute name is free-form (convention: leading
  underscore, e.g. `_unique_name`) — it is not looked up by that name, only its value
  matters. Real examples in core 19.0: `addons/project/models/project_tags.py`
  (`_name_uniq`), `addons/project/models/project_project.py` (`_project_date_greater`).

## Verification

After removing `_()` from field definitions, switch the UI language to Arabic (or any non-English language) and verify that field labels translate correctly from the `.po` file.

## References

- [Odoo ORM Fields Documentation](https://www.odoo.com/documentation/19.0/developer/reference/backend/orm.html#fields)
- Fixed in: `custom/customer_level_chart/models/customer_level.py`
- Fixed in: `custom/customer_level_chart/models/res_partner.py`
- Fixed in: `custom/stock_valuation_report/models/stock_move_valuation.py`
- Fixed in: `custom/delivery_vehicle/models/delivery_vehicle.py`

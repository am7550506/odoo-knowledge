# N+1 Query Problem in Computed Fields — Use `read_group()` Instead

| Field         | Value                          |
|---------------|--------------------------------|
| Category      | performance                    |
| Odoo Versions | All (14, 15, 16, 17, 18, 19)  |
| Severity      | 🔴 Critical                    |
| Last Verified | 2026-05-30                     |
| Author        | ENG/Gamal Mansour              |

**Tags:** `performance`, `N+1`, `computed-fields`, `read_group`, `search`, `sql`

---

## Problem

A computed field that calls `search()` or `search_count()` inside a loop causes N+1 SQL queries — one query per record:

```python
# ❌ WRONG — N separate SQL queries (one per target)
@api.depends('salesperson_id', 'date_from', 'date_to')
def _compute_achievement(self):
    for target in self:
        invoices = self.env['account.move'].search([
            ('invoice_user_id', '=', target.salesperson_id.id),
            ('invoice_date', '>=', target.date_from),
            ('invoice_date', '<=', target.date_to),
        ])
        target.achieved_amount = sum(invoices.mapped('amount_total'))
```

With 100 records open → 100 SQL queries fired simultaneously.

## Root Cause

Each `search()` call inside a `for record in self` loop issues a separate database query. Odoo computes fields for all records in a single batch (`self` is a recordset), so the fix is to fetch all data in **one aggregated query** and distribute.

## Solution ✅

Use `read_group()` to aggregate data in a single SQL query, then distribute results using a dict lookup:

```python
# ✅ CORRECT — 1 aggregated SQL query regardless of N records
@api.depends('salesperson_id', 'date_from', 'date_to')
def _compute_achievement(self) -> None:
    valid = self.filtered(lambda t: t.salesperson_id and t.date_from and t.date_to)
    # Zero-out invalid
    (self - valid).write_or_set_defaults(...)

    if not valid:
        return

    all_user_ids = valid.mapped('salesperson_id').ids
    min_date = min(valid.mapped('date_from'))
    max_date = max(valid.mapped('date_to'))

    groups = self.env['account.move'].read_group(
        domain=[
            ('invoice_user_id', 'in', all_user_ids),
            ('invoice_date', '>=', min_date),
            ('invoice_date', '<=', max_date),
            ('state', '=', 'posted'),
            ('move_type', '=', 'out_invoice'),
        ],
        fields=['invoice_user_id', 'amount_total:sum'],
        groupby=['invoice_user_id', 'invoice_date:month'],
        lazy=False,
    )

    # Build lookup dict from results
    from collections import defaultdict
    amount_map = defaultdict(float)
    for group in groups:
        user_id = group['invoice_user_id'][0]
        month_key = group.get('invoice_date:month', '')
        amount_map[(user_id, month_key)] += group.get('amount_total', 0.0)

    # Distribute to each record — O(1) lookup per record
    for target in valid:
        key = (target.salesperson_id.id, f"{target.year}-{target.month}")
        target.achieved_amount = amount_map.get(key, 0.0)
```

## ⚠️ Pitfalls

- `read_group` with `:month` groupby returns keys like `'2025-05'` (YYYY-MM) — not a date object.
- **Many2many computed fields** cannot be populated from `read_group` — still need `search()` for those. Try to minimize their use in list views.
- If date ranges differ per record (like monthly targets), use `min_date`/`max_date` as the outer boundary and filter in Python — still O(1) queries.
- `lazy=False` is required to get all groupby levels resolved.

## Verification

Enable SQL logging and compare query count before/after:
```python
import logging
logging.getLogger('odoo.sql_db').setLevel(logging.DEBUG)
```

## References

- [Odoo read_group docs](https://www.odoo.com/documentation/19.0/developer/reference/backend/orm.html#odoo.models.Model.read_group)
- Fixed in: `custom/sale_target/models/sale_target.py`

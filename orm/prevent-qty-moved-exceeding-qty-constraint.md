# Preventing Released/Stacked Quantity from Exceeding Actual Quantity

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | orm                                        |
| Odoo Versions | 18                                         |
| Severity      | 🔴 Critical                                |
| Last Verified | 2026-05-20                                 |
| Author        | ENG/Mohamed Saber                          |

**Tags:** `orm`, `constraint`, `validation`, `related-field`

---

## Problem

In warehouse handling modules (e.g., `inland_container_terminal`), transactions of type `Stack` or `Release` allow setting a "Qty Moved" (`qty_moved`). However, due to multiple design flaws, the quantity stacked/released can exceed the actual quantity of the parent commodity operation:

1. **Stale Validation Data**: The constraint validation check compares `qty_moved` against `remaining_qty`. However, `remaining_qty` is a static Float field updated only via client-side `@api.onchange` when the commodity operation is selected. If the record is created programmatically, or other fields are edited, this value is outdated, allowing incorrect saves.
2. **Conflicting Related Fields**: The `qty` field is defined as a related field to `commodity_operation_id.qty` with `readonly=False`. If the user reduces this quantity after confirmation, it updates the parent commodity operation without triggering the `qty_moved` validations.
3. **Confirmed Records Editable**: Confirming the transaction does not lock the fields in the UI, allowing users to manually increase `qty_moved` or reduce `qty` without updating computed totals on the parent commodity operation.

## Root Cause

1. Relying on an `@api.onchange` to populate validation fields used in `@api.constrains` rather than computing validation values dynamically on the server.
2. Returning a `warning` dictionary inside a python constraint (which Odoo ignores during database write/create) instead of raising `ValidationError`.
3. Lack of database-level constraints on the parent model (`commodities.operations`) to prevent its quantity (`qty`) from being reduced below the quantity already moved (`qty_moved`).
4. Absence of UI-level locks (`readonly="is_confirm"`) on fields of confirmed records.

## Solution ✅

### 1. Dynamic Validation on the Handling Model
Modify the constraint on `warehouse.handling` to compute the total stacked quantity dynamically from the database:

```python
    @api.constrains('commodity_operation_id', 'qty_moved', 'qty')
    def check_data_container_number(self):
        for rec in self:
            if rec.service_class_id.handling_purpose == 'Stack':
                if rec.qty_moved <= 0:
                    raise ValidationError(_("يجب ان تكون الكمية المراد تخزينها اكبر من الصفر"))
                
                domain = [
                    ('commodity_operation_id', '=', rec.commodity_operation_id.id),
                    ('service_class_id.handling_purpose', '=', 'Stack'),
                    ('id', '!=', rec.id),
                ]
                other_handlings = self.env['warehouse.handling'].search(domain)
                total_other_moved = sum(other_handlings.mapped('qty_moved'))
                
                if total_other_moved + rec.qty_moved > rec.commodity_operation_id.qty:
                    raise ValidationError(_("الكمية المراد تخزينها أكبر من الكمية الاجمالية المتاحة في أمر البضاعة (%s > %s)") % (
                        round(total_other_moved + rec.qty_moved, 2),
                        round(rec.commodity_operation_id.qty, 2)
                    ))
```

### 2. Constraint on the Parent Model
Add a validation constraint on `commodities.operations` so that its `qty` cannot be set lower than its `qty_moved`:

```python
    @api.constrains('qty', 'qty_moved')
    def _check_qty_moved_limit(self):
        for rec in self:
            if rec.qty < rec.qty_moved:
                raise ValidationError(_("الكمية الكلية لا يمكن أن تكون أقل من الكمية التي تم نقلها بالفعل (%s < %s)") % (
                    round(rec.qty, 2),
                    round(rec.qty_moved, 2)
                ))
```

### 3. Making Confirmed Records Readonly in XML
Apply `readonly="is_confirm"` to fields in `warehouse_handling_view.xml` so users cannot modify confirmed transactions.

## ⚠️ Pitfalls

- **Bypassing UI logic**: Do not write validations that rely on non-stored computed fields or UI-only fields that do not update in backend `write`/`create`.
- **Warning Dicts in Constraints**: Never use `return {'warning': ...}` in Odoo `@api.constrains`. It does not raise an exception or abort the transaction. Always use `raise ValidationError(_(...))`.

## Verification

When trying to save a transaction where `qty_moved` exceeds the available quantity, Odoo raises a validation error block. When trying to edit `qty` on a confirmed record or reducing the commodity operation's `qty` below `qty_moved`, the save operation is rejected with a clean localized error.

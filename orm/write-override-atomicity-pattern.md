# write() Override — State Changes Must Come AFTER super().write()

| Field         | Value                          |
|---------------|--------------------------------|
| Category      | orm                            |
| Odoo Versions | All (14, 15, 16, 17, 18, 19)  |
| Severity      | 🔴 Critical                    |
| Last Verified | 2026-05-30                     |
| Author        | ENG/Gamal Mansour              |

**Tags:** `write`, `override`, `atomicity`, `state-machine`, `transaction`

---

## Problem

When overriding `write()`, changing state on related records **before** calling `super().write()` can cause data corruption if the DB write fails:

```python
# ❌ WRONG — state changed before write, not atomic
def write(self, vals):
    if 'vehicle_id' in vals:
        vehicle = self.env['delivery.vehicle'].browse(vals['vehicle_id'])
        vehicle.state = 'assigned'   # ← happens even if write() fails below!

    return super().write(vals)       # ← if this raises, vehicle.state is corrupt
```

If `super().write()` fails (e.g. SQL constraint violation, concurrent update), the vehicle state has already been modified with no rollback.

## Root Cause

Odoo's ORM runs inside a database transaction, but Python-level code that mutates other records **before** the main write is not automatically rolled back if an exception occurs later in the same method. The DB itself will rollback, but only if the exception propagates to the top-level transaction boundary.

In practice, partial state corruption can happen when `super().write()` raises a `UserError` or `ValidationError` that is caught somewhere in the call stack.

## Solution ✅

Always perform side-effect state changes **after** `super().write()`:

```python
# ✅ CORRECT — all state changes happen after the write succeeds
def write(self, vals: dict) -> bool:
    """Sync related model state after successful write."""
    res = super().write(vals)  # ← DB write happens first

    # State changes here — only reached if super() succeeded
    if 'vehicle_id' in vals and vals['vehicle_id']:
        vehicle = self.env['delivery.vehicle'].browse(vals['vehicle_id'])
        if vehicle.state == 'available':
            vehicle.state = 'assigned'

    if 'state' in vals and vals['state'] == 'done':
        for record in self:
            # update related records...
            pass

    return res
```

## ⚠️ Pitfalls

- If you need the **old** value of a field before the write, capture it before `super()`:
  ```python
  old_states = {r.id: r.state for r in self}
  res = super().write(vals)
  # use old_states[r.id] after super()
  ```
- Don't confuse this with `create()` — in `create()`, `super()` must come first by definition.
- This pattern applies equally to `unlink()` overrides.

## Verification

Write a test that triggers a `_sql_constraints` violation in `super().write()` and verify that no side-effect state changes occurred on related records.

## References

- Fixed in: `custom/delivery_vehicle/models/stock_picking_batch.py`

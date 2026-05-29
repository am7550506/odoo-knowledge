# crm.lead Mobile Field Removal in Odoo 19

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | upgrade                                    |
| Odoo Versions | 19                                         |
| Severity      | 🔴 Critical                                |
| Last Verified | 2026-05-25                                 |
| Author        | ENG/Mohamed Hamdy                          |

**Tags:** `upgrade`, `migration`, `crm.lead`, `mobile`, `field`

---

## Problem

> When trying to create or write a lead in Odoo 19, custom code that accesses or constrains `lead.mobile` will throw an AttributeError because the `mobile` field has been removed from the `crm.lead` model.

```
AttributeError: 'crm.lead' object has no attribute 'mobile'
```

## Root Cause

> The `mobile` field was removed from the standard `crm.lead` model in Odoo 19. All phone numbers should be stored in the `phone` field, or the `mobile` field from the related `res.partner` should be used instead.

## Solution ✅

> Remove all references to `mobile` in your custom `crm.lead` models, constraints, and views.

```python
# Before
@api.constrains('phone', 'mobile')
def _check_phone_mobile_format(self) -> None:
    for lead in self:
        # ...
        if lead.mobile and not pattern.match(lead.mobile):
            raise ValidationError(_('Invalid Mobile'))

# After
@api.constrains('phone')
def _check_phone_format(self) -> None:
    for lead in self:
        # ...
        # (Remove lead.mobile check entirely)
```

## ⚠️ Pitfalls

- Forgetting to remove `mobile` from XML form views, tree views, or search views will cause view parsing errors on load.
- If you rely on `mobile` for business logic (e.g., SMS sending), you may need to fallback to `phone` or `partner_id.mobile`.

## Verification

> Creating or updating a lead through the frontend UI or API will no longer throw the AttributeError related to the `mobile` field.

# AccessError: You are not allowed to access 'Currency' (res.currency) records during QWeb rendering

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | security                                   |
| Odoo Versions | 15, 16, 17, 18, 19 (or "All")             |
| Severity      | 🔴 Critical                                |
| Last Verified | 2026-05-20                                 |
| Author        | ENG/Mohamed Hamdy                          |

**Tags:** `security`, `access-rights`, `public-user`, `portal-user`, `qweb`, `res.currency`

---

## Problem

When loading the Odoo website or login page (anonymous frontend session), Odoo crashes with a `QWebException` / `AccessError` during the rendering of the `web.frontend_layout` template:

```
odoo.addons.base.models.ir_qweb.QWebException: Error while render the template
AccessError: You are not allowed to access 'Currency' (res.currency) records.

This operation is allowed for the following groups:
        - Accounting/Administrator
        - Administration/Settings
        - User types/Internal User
        - User types/Portal
        - User types/Public

Contact your administrator to request access if necessary.
Template: web.frontend_layout
Path: /t/html/head/script[2]/t
Node: <t t-out="json.dumps(request.env[\'ir.http\'].get_frontend_session_info())"/>
```

## Root Cause

1. The anonymous/public sessions in Odoo frontend run under the **Public User** (`base.public_user`).
2. If the **Public User** record is deactivated (`active = False`), or loses its standard group **User types / Public** (`base.group_public`), it will not have permission to read the default currency (`res.currency`) when generating the frontend session info (which requires retrieving the company currency).
3. Without this permission, loading the login or frontend portal pages throws a critical `AccessError` and blocks access.

## Solution ✅

Use a Python script or Odoo shell to reactivate the Public User and reassign it to the **User types / Public** group:

```python
# Save this as a scratch script or run in Odoo shell
import odoo
from odoo import api, SUPERUSER_ID

db_name = 'your_database_name'
registry = odoo.registry(db_name)

with registry.cursor() as cr:
    env = api.Environment(cr, SUPERUSER_ID, {})
    
    # 1. Fetch Public User (base.public_user)
    public_user = env.ref('base.public_user', raise_if_not_found=False)
    if not public_user:
        public_user = env['res.users'].search([('login', '=', 'public')], limit=1)
        
    if public_user:
        # Reactivate user
        if not public_user.active:
            public_user.write({'active': True})
            
        # Reassign to "User types / Public" group (base.group_public)
        group_public = env.ref('base.group_public')
        if group_public not in public_user.groups_id:
            public_user.write({'groups_id': [(4, group_public.id)]})
            
        cr.commit()
        print("Public User successfully activated and assigned to Public group.")
```

## ⚠️ Pitfalls

- **Do NOT deactivate the Public User**: The public user must remain active for Odoo to render frontend pages for guests.
- **Do NOT remove base.group_public from Public User**: The frontend session calls methods that expect the public user to have read permissions on several basic models (like `res.currency`, `res.company`, etc.).

## Verification

To verify that the fix worked:
1. Log out or open a private browser window.
2. Visit the Odoo home page or login screen.
3. The page should load successfully without throwing `AccessError`.

## References

- Related file: `security/public-user-inactive-res-currency-accesserror.md`

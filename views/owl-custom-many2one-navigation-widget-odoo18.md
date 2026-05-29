# 🧩 OWL Custom Many2one Navigation Widget in Odoo 18

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | views                                      |
| Odoo Versions | 17, 18, 19                                 |
| Severity      | 🟡 Medium                                  |
| Last Verified | 2026-05-20                                 |
| Author        | ENG/Mohamed Saber                          |

**Tags:** `owl`, `javascript`, `many2one`, `widget`, `navigation`, `view`

---

## Problem

Sometimes we want to render a Many2one field in a Tree/List view as a clickable, lightweight hyperlink that opens the related document's form view inside the standard Odoo breadcrumbs, instead of using standard Odoo relational popups or navigation clicks.

Odoo has a standard widget `open_move_widget` for `account.move` but doesn't have equivalents for other relational fields (like `partner_id` or `product_id`).

## Solution ✅

Create a dedicated frontend-only OWL component and register it in the standard `fields` registry.

### 1. OWL Javascript Component Definition

Create `static/src/components/your_widget_name/your_widget_name.js`:

```javascript
/** @odoo-module **/

import { registry } from "@web/core/registry";
import { useService } from "@web/core/utils/hooks";
import { standardFieldProps } from "@web/views/fields/standard_field_props";
import { Component } from "@odoo/owl";

class OpenPartnerWidget extends Component {
    static template = "partner_open_widget.OpenPartnerWidget";
    static props = { ...standardFieldProps };

    setup() {
        super.setup();
        this.action = useService("action");
    }

    get partnerId() {
        const val = this.props.record.data[this.props.name];
        if (Array.isArray(val)) {
            return val[0];
        }
        return val && typeof val === "object" ? val.resId : val;
    }

    get partnerName() {
        const val = this.props.record.data[this.props.name];
        if (Array.isArray(val)) {
            return val[1];
        }
        return val && typeof val === "object" ? val.displayName : (val || "");
    }

    async openPartner() {
        const partnerId = this.partnerId;
        if (partnerId) {
            this.action.doAction({
                type: "ir.actions.act_window",
                res_model: "res.partner",
                res_id: partnerId,
                views: [[false, "form"]],
                target: "current",
            });
        }
    }
}

registry.category("fields").add("open_partner_widget", {
    component: OpenPartnerWidget,
});
```

### 2. XML QWeb Template

Create `static/src/components/your_widget_name/your_widget_name.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<templates xml:space="preserve">
    <t t-name="partner_open_widget.OpenPartnerWidget">
        <t t-if="partnerId">
            <a href="#" t-out="partnerName" t-on-click.prevent.stop="(ev) => this.openPartner()"/>
        </t>
        <t t-else="">
            <span t-out="partnerName"/>
        </t>
    </t>
</templates>
```

### 3. Manifest Registration

Register both assets under `'web.assets_backend'` bundle in `__manifest__.py`:

```python
    'assets': {
        'web.assets_backend': [
            'partner_open_widget/static/src/components/open_partner_widget/open_partner_widget.js',
            'partner_open_widget/static/src/components/open_partner_widget/open_partner_widget.xml',
        ],
    },
```

---

## ⚠️ Pitfalls

- **Relational Field Formats:** Remember that relational Many2one values in Odoo 18 can be returned as an array `[id, name]` or as an active recordset object. Always parse both conditions dynamically inside your JS getters.
- **Empty values:** If the field value is `False`, do not render the anchor tag to avoid broken clicks and standard Javascript exceptions.

## Verification

Update your custom module and check that the target field displays as a beautiful blue clickable link that smoothly redirects to the related record upon click.

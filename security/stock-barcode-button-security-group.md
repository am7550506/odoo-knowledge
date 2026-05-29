# Adding Security Group Access Rights to Stock Barcode OWL Buttons

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | security                                   |
| Odoo Versions | 17, 18, 19                                 |
| Severity      | 🟢 Low                                    |
| Last Verified | 2026-05-24                                 |
| Author        | ENG/Mohamed Saber                          |

**Tags:** `security`, `owl`, `stock_barcode`, `access-rights`

---

## Problem

> You need to hide a specific button (e.g., "+ ADD PRODUCT") inside the Odoo Enterprise Stock Barcode app (`stock_barcode`) based on a specific user security group.
> Since it's an OWL component and rendered on the frontend, standard backend XML `groups="..."` attributes do not work the same way as they do in form views.

## Root Cause

> The Stock Barcode app (`MainComponent`) receives its state and configuration (including security groups) via an RPC call to `/stock_barcode/get_barcode_data` before mounting. To use a custom group in the XML template (`t-if="groups.group_name"`), the group's boolean evaluation must be passed from the backend controller to the frontend JS.

## Solution ✅

> **Step 1: Define the Security Group**
> Create your standard `res.groups` record in XML.

```xml
<record id="group_stock_barcode_add_product" model="res.groups">
    <field name="name">Stock Barcode: Add Product</field>
    <field name="category_id" ref="base.module_category_inventory_inventory"/>
</record>
```

> **Step 2: Pass Group Status via Controller Override**
> Inherit `StockBarcodeController` and override `_get_groups_data` to inject your custom group.

```python
from odoo.http import request
from odoo.addons.stock_barcode.controllers.stock_barcode import StockBarcodeController

class StockBarcodeControllerInherit(StockBarcodeController):
    def _get_groups_data(self):
        groups_data = super()._get_groups_data()
        groups_data['group_stock_barcode_add_product'] = request.env.user.has_group('your_module.group_stock_barcode_add_product')
        return groups_data
```

> **Step 3: Use the Group in the OWL Template**
> Use `xpath` to inherit `stock_barcode.MainComponent` and add a `t-if` reading from `groups`.

```xml
<templates id="template" xml:space="preserve">
    <t t-inherit="stock_barcode.MainComponent" t-inherit-mode="extension">
        <xpath expr="//button[hasclass('o_add_line')]" position="attributes">
            <attribute name="t-if">groups.group_stock_barcode_add_product</attribute>
        </xpath>
    </t>
</templates>
```

## ⚠️ Pitfalls

- **Do NOT** try to use standard backend `groups="module.group_id"` inside `t-if` expressions in OWL templates, as they evaluate in the JS context and lack backend session context.
- Ensure your `xpath` targets the correct button class (e.g., `hasclass('o_add_line')`) since class names might change slightly between Odoo versions.

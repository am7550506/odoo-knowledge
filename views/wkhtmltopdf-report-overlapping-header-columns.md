# Overlapping Header Columns in Wkhtmltopdf QWeb Reports

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | views                                      |
| Odoo Versions | 17, 18, 19                                 |
| Severity      | 🟡 Medium                                  |
| Last Verified | 2026-05-20                                 |
| Author        | ENG/Mohamed Saber                          |

**Tags:** `wkhtmltopdf`, `qweb`, `report`, `bootstrap`, `overlap`, `css`, `col-auto`

---

## Problem

When printing custom QWeb reports in newer Odoo versions (V17+ / V18+), columns inside `<div id="informations" class="row">` overlap, run into each other, or bundle up horizontally on a single line.

Example overlapping text in the header:
```
Invitation Receiving Date:Payment Terms:Price List:
01/11/2023            15 Days       Public Pricelist (USD)
```

## Root Cause

1. Newer Odoo versions use **Bootstrap 5**.
2. Legacy reports use column classes like `class="col-auto mw-100 mb-2"` inside the `informations` grid.
3. Wkhtmltopdf's rendering engine has issues calculating flexible dimensions with `col-auto` and `mw-100` under newer CSS frameworks, leading to columns collapsing to zero width or overlapping each other.

## Solution ✅

Replace the legacy `col-auto mw-100 mb-2` column classes with standard Bootstrap column classes like `col` (equal-width columns) or explicit grid classes like `col-3` or `col-4`.

### Before (Buggy):
```xml
<div id="informations" class="row mt32 mb32">
    <div t-if="doc.reference_no" class="col-auto mw-100 mb-2">
        <strong>Reference NO:</strong>
        <p t-field="doc.reference_no" class="m-0" />
    </div>
    <div t-if="doc.invitation_deceiving_date" class="col-auto mw-100 mb-2">
        <strong>Invitation Receiving Date:</strong>
        <p t-field="doc.invitation_deceiving_date" class="m-0" />
    </div>
</div>
```

### After (Fixed & Premium Layout):
```xml
<div id="informations" class="row mb-4">
    <div t-if="doc.reference_no" class="col" name="informations_reference">
        <strong>Reference NO:</strong>
        <p t-field="doc.reference_no" class="m-0" />
    </div>
    <div t-if="doc.invitation_deceiving_date" class="col" name="informations_invitation">
        <strong>Invitation Receiving Date:</strong>
        <p t-field="doc.invitation_deceiving_date" class="m-0" />
    </div>
</div>
```

*Note: In Odoo 18, standard reports (like sales and invoices) use `class="col"` for each parameter column under `#informations`.*

## ⚠️ Pitfalls

- **Do NOT use col-auto on reports**: Always prefer `col` (for equal fluid widths) or `col-3` (to display up to 4 items per row) inside printing templates. Wkhtmltopdf behaves unpredictably with `col-auto`.

## Verification

Run an upgrade on the custom module and reprint the document:
```bash
./venv/bin/python3 ./odoo-bin -d your_db -u your_module --stop-after-init
```
Verify the columns are nicely separated and proportional.

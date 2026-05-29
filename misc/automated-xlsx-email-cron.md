# Automated XLSX Report Email via Cron in Odoo 18

| Field         | Value                                      |
|---------------|---------------------------------------------|
| Category      | misc                                        |
| Odoo Versions | 17, 18                                      |
| Severity      | 🟡 Medium                                   |
| Last Verified | 2026-05-24                                  |
| Author        | ENG/Mohamed Saber (via Antigravity)          |

**Tags:** `xlsx`, `cron`, `email`, `report`, `pos`, `scheduled-action`, `ir.config_parameter`

---

## Problem

> Need to automatically generate an XLSX report from POS order line data and send it via email on a schedule (e.g., weekly). The RAMA project (Odoo 17) had this feature with hardcoded database-specific IDs, which breaks when porting to another database or Odoo version.

## Root Cause

> Hardcoding database-specific IDs (POS config IDs, brand IDs, company ID) in the report generation logic makes the feature non-portable across databases. Additionally, Odoo version differences (v17 `detailed_type` vs v18 `type`) require field mapping adjustments.

## Solution ✅

### Architecture

1. **Inherit `pos.order.line`** to add `get_report_base_filename()` and `action_send_line_report_email()` methods.
2. **Create an AbstractModel** (`report.module_name.report_name`) inheriting `report.report_xlsx.abstract` to generate the XLSX.
3. **Use `ir.config_parameter`** (System Parameters) for all configurable filters:
   - `module_name.xlsx_pos_config_ids` — comma-separated POS config IDs
   - `module_name.xlsx_brand_ids` — comma-separated brand IDs
   - `module_name.xlsx_company_id` — company ID
   - `module_name.xlsx_days_back` — days back from today
4. **Define in XML:**
   - `ir.actions.report` with `report_type=xlsx`
   - `mail.template` with `report_template_ids` linking to the report
   - `ir.cron` calling `model.action_send_line_report_email()`
   - Default `ir.config_parameter` records

### Key Code Pattern

```python
# In the cron method:
def action_send_line_report_email(self):
    template = self.env.ref('module.template_xml_id', raise_if_not_found=False)
    line = self.env['pos.order.line'].sudo().search([], limit=1)
    if line and template:
        template.send_mail(line.id, force_send=True)
```

```python
# In the report class:
def _get_report_lines(self):
    ICP = self.env['ir.config_parameter'].sudo()
    raw = ICP.get_param('module.xlsx_pos_config_ids', '')
    if raw and raw.strip():
        ids = [int(x.strip()) for x in raw.split(',') if x.strip()]
        domain.append(('field', 'in', ids))
```

## ⚠️ Pitfalls

- **Hardcoded IDs:** NEVER hardcode database-specific IDs. Use `ir.config_parameter` for portability.
- **Cron Active State:** Set `<field name="active" eval="False"/>` by default to prevent accidental emails during install.
- **Odoo 18 Field Changes:** `product.detailed_type` is renamed to `product.type` in v18.
- **Email Template `email_to`:** Must be manually configured post-install or emails won't be sent.
- **Performance:** Use proper domain filtering to avoid loading millions of POS lines.
- **`report_xlsx` dependency:** The `report_xlsx` module must be installed (community module providing `report.report_xlsx.abstract`).

## Verification

```bash
# 1. Upgrade the module
odoo-bin -d <db> -u pos_order_line_report --stop-after-init

# 2. Check cron was created
# Settings → Technical → Scheduled Actions → "send POS Order Line XLSX"

# 3. Configure parameters
# Settings → Technical → System Parameters → set filter values

# 4. Set email recipient
# Settings → Technical → Email Templates → "POS Order Line XLSX" → set email_to

# 5. Test manually
# Scheduled Actions → "send POS Order Line XLSX" → "Run Manually"
```

## References

- RAMA project source: `/home/mohamed/odoo/odoo17/RAMA/pos_order_line_report/models/pos_order_line_xlsx.py`
- WAMA implementation: `/home/mohamed/odoo/odoo18new/WAMA-Group/pos_order_line_report/models/pos_order_line_xlsx.py`
- Odoo `report_xlsx` docs: https://github.com/OCA/reporting-engine/tree/18.0/report_xlsx

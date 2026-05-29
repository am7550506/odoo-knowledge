# Include Waiting (Verify) Payslips in Master Report

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | orm                                        |
| Odoo Versions | 16, 17, 18, 19                             |
| Severity      | 🟢 Low                                    |
| Last Verified | 2026-05-22                                 |
| Author        | ENG/Mohamed Saber                          |

**Tags:** `orm`, `hr_payroll`, `master-report`, `payslip`, `localization`

---

## Problem

> In Odoo's localizations (e.g. `l10n_eg_hr_payroll`, `l10n_sa_hr_payroll`), the Master Report strictly searches for payslips that are in `done` or `paid` state. If users want to print the report to review the payslips while they are still in the `verify` (Waiting) state, they get a validation error: "There are no eligible payslips for that period of time".

## Root Cause

> The `action_generate_report` and `_compute_period_has_payslips` methods in `report.l10n_eg_hr_payroll.master` (and similar localization models) have hardcoded search domains: `[("state", "in", ["done", "paid"])]`.

## Solution ✅

> Create a custom module to inherit the model and override these methods to include the `verify` state.

```python
# models/hr_payroll_master_report.py
from odoo import models, api, _
from odoo.exceptions import ValidationError, UserError
import base64
import io
from collections import defaultdict
from datetime import datetime
from odoo.tools.misc import xlsxwriter

# Must redefine or import XLSX constants from original file
XLSX = {
    "NUMBER": 0,
    "TEXT": 1,
    "DATE": 2,
    "FORMULA": 3,
    "LABEL": 4,
}

class HrEgMasterReport(models.Model):
    _inherit = "report.l10n_eg_hr_payroll.master"

    @api.depends("date_from", "date_to")
    def _compute_period_has_payslips(self):
        for report in self:
            payslips = report.env["hr.payslip"].search([
                ("date_from", ">=", report.date_from),
                ("date_to", "<=", report.date_to),
                ("company_id", "=", report.env.company.id),
                # Added 'verify' to the domain
                ("state", "in", ["verify", "done", "paid"]),
            ])
            report.period_has_payslips = bool(payslips)

    def action_generate_report(self):
        # Override completely because the search domain is hardcoded
        # (Copy the original logic and change the state domain)
        # ... [original logic]
        payslips = self.env["hr.payslip"].search([
            ("date_from", ">=", self.date_from),
            ("date_to", "<=", self.date_to),
            ("company_id", "=", company.id),
            ("state", "in", ["verify", "done", "paid"]), # Modified here
        ])
        # ... [rest of original logic]
```

## ⚠️ Pitfalls

- **Uncomputed Payslips:** Payslips in the `verify` state might not have their `line_ids` fully computed or accurate if the user edited inputs but didn't hit "Compute Sheet". This might result in a Master Report with missing/wrong numbers.
- **Copy-Paste Maintenance:** Since we are overriding the entire `action_generate_report` method, any future upstream updates from Odoo to the report formatting will not be inherited automatically.

## Verification

> Generate the Master Report for a period containing only `verify` (Waiting) payslips. It should successfully download the Excel file.

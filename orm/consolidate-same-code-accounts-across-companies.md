# Merging Accounts That Share a Code Across Companies for a Consolidation Report

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | orm (custom reporting wizard)              |
| Odoo Versions | 19 (relies on `code_store` being a jsonb keyed by company_id — verify on 17/18 where `code` may be a plain Char) |
| Severity      | 🟡 Medium |
| Last Verified | 2026-09-22 |
| Author        | ENG/Gamal Mansour |

**Tags:** `consolidation`, `multi-company`, `multi-currency`, `account.account`, `account.group`, `raw-sql`, `wizard`

---

## Problem

Follow-up to [[multicompany-consolidated-report-currency-follows-active-company]].
Client had two companies (FZCO, LLC), each with its **own separate `account.account`
record** named "Computers Dep." and coded `70011` — not one shared account via
`account_account_res_company_rel`. Wanted a report where: pick N companies, and
accounts sharing the same code across those companies are summed into **one** line,
grouped under their Account Group, converted into one currency the user picks — with
the total NOT depending on which company happens to be active (the root cause from the
linked entry).

## Root Cause / Design Constraint

- In Odoo 19, `account.account.code` is **not** a plain column — it's extracted from a
  jsonb `code_store` field keyed by `company_id` (as a string), via
  `with_company(...)._field_to_sql(...)`. Two unrelated `account.account` rows can have
  the *same code value* under different companies purely by coincidence (same CoA
  template), with no FK linking them.
- `account.account.group_id` is a **non-stored compute** (`_compute_account_group`),
  resolved by matching the company-scoped `code` against `account.group` prefix ranges —
  and `account.group` itself is **per-company** (`company_id` required field). So there
  is no single shared "group" row to point multiple companies' accounts at; you can only
  resolve one company's group as a representative for display.

## Solution ✅

Built a standalone module (`ayadi_consolidation_report`) with a `TransientModel` wizard:

1. Wizard fields: `company_ids` (m2m res.company, domain limited to
   `self.env.user.company_ids` — not `self.env.companies`, so it isn't limited to
   whichever companies happen to be checked in the switcher), `date_from`, `date_to`,
   `target_currency_id`.
2. One raw SQL query (justified: the group-by key, the account code, is a jsonb lookup
   keyed by the row's own `company_id` — `read_group` cannot express that):
   ```sql
   SELECT account_move_line.company_id,
          account_account.id AS account_id,
          account_account.code_store->>(account_move_line.company_id::text) AS account_code,
          COALESCE(account_account.name->>%(lang)s, account_account.name->>'en_US') AS account_name,
          SUM(account_move_line.balance) AS balance
     FROM account_move_line
     JOIN account_account ON account_account.id = account_move_line.account_id
    WHERE account_move_line.company_id = ANY(%(company_ids)s)
      AND account_move_line.parent_state = 'posted'
      AND account_move_line.date BETWEEN %(date_from)s AND %(date_to)s
 GROUP BY account_move_line.company_id, account_account.id, account_code, account_name
   ```
   (`parent_state` is a real stored column on `account_move_line` in this version —
   verified against the live schema, it is not a related/computed-only field here.)
3. In Python, merge the rows **by `account_code`** across companies. For each merged
   code, convert every contributing company's native amount using
   `company.currency_id._get_conversion_rate(company.currency_id, target_currency,
   company, date_to)` — **passing that row's own `company`**, never `self.env.company` —
   which is exactly what avoids the "total changes with the active company" bug from the
   linked entry, because the rate source is now pinned per-line instead of ambient.
4. Resolve the merged line's `group_id` from **one representative account**
   (`account.browse(id).with_company(that_companys_company)`).group_id — accept as a
   known limitation if two companies file the same code under genuinely different
   groups (flag it as a chart-of-accounts inconsistency to fix at the source, don't try
   to silently reconcile it in the report).
5. Keep a human-readable "per-company breakdown" string (native amount + converted
   amount per company) on each result line — critical for the accountant to audit the
   total back to source without re-deriving the FX math by hand.

## Follow-up: rate per transaction date + month-over-month comparison

Client's next ask: don't use one flat rate for the whole period - convert each
transaction at the rate valid **on its own date** - and let them compare the
consolidated total **month by month**. Both fit the same design cleanly:

- Group the raw SQL by the **exact `account_move_line.date`** (not just by
  account/company) instead of collapsing straight to one SUM per account. Then
  in Python, cache `_get_conversion_rate(company.currency_id, target, company,
  that_exact_date)` per `(company_id, date)` pair (a plain dict cache — avoids
  re-querying the rate for every line that shares a date) and convert each
  day's bucket individually before summing into the code's total.
- Bucket the converted amounts by `date.replace(day=1)` ("period") and create
  **one `consolidation.report.line` per (code, period)** instead of one per
  code overall. A `period` Date field on the line model plus a **Pivot view**
  (`<field name="period" type="col" interval="month"/>`, rows = group/code/
  name, measure = balance) gives the month-over-month comparison for free —
  no extra wizard toggle needed. The List view (grouped by group/code) still
  gives the plain total when the month breakdown isn't wanted.
- **Verified live** against the project's real `ayadi_test` database via
  `odoo-bin shell` (not just synthetic unit tests): the same FZCO monthly AED
  depreciation entry (constant 1019.38 AED every month) converted to a
  **different** EGP amount each month (13000.76 → 13280.58 → 14861.13 → ...),
  proving the per-date rate lookup is real and not silently falling back to
  one rate. Re-running with the active company flipped from LLC to FZCO
  produced byte-for-byte identical monthly totals — the original fix still
  holds after this change.

## Follow-up: pivot row labels rendering blank / duplicate-looking rows

Client reported the Pivot view's row header column showing **blank text** and
what looked like duplicate rows with identical figures, right after the
month-comparison feature above shipped. The underlying `consolidation.report.
line` data was verified correct via direct SQL (no duplicate `(code, period)`
rows for the wizard run in question) — this was a **view-level** issue, not a
data bug.

Root cause: the pivot declared **3 separate row levels**
(`group_id` -> `code` -> `name`), all as plain fields, with hundreds of
distinct `code`/`name` combinations. Odoo's pivot widget is built for a
moderate number of distinct values per level with a clean drill-down; feeding
it two more high-cardinality Char levels on top of a Many2one level is not
standard usage and produced an unreadable/blank-looking result.

**Fix:** collapse the two leaf levels into **one combined Char field**
(`account_label`, e.g. `"70011 Computers Dep."`, built once in Python when the
line is created: `f"{code} {name}"`) and use only **two** pivot row levels:
`group_id` (Many2one, moderate cardinality) then `account_label` (one leaf per
account, no further drill needed since code and name are 1:1 already). Kept
`code`/`name` as separate stored fields for the List view's own columns —
only the *pivot's row axis* needed simplifying.

## ⚠️ Pitfalls

- Don't assume two accounts with the same name/code across companies are the same
  `account.account` record — check `account_account_res_company_rel` before writing any
  code that assumes a shared account. In this project they were separate rows.
- `account.group` is per-company; there is no "give me the group that applies to all
  these companies" query. Pick a representative and say so, don't fake a merge.
- Access rights: `account.group_account_readonly` already has a company-agnostic
  `ir.rule` on `account.move.line` in core (`account_move_line_rule_group_readonly`,
  domain `(1,=,1)`) — gate the new wizard/lines models on that same group and the
  cross-company read works without `sudo()` or `allowed_company_ids` context tricks.
- This method uses one spot rate per company (as of `date_to`), not the
  historical/average/current CTA method core P&L uses for equity vs. income/expense
  accounts. Fine for a quick consolidated total; say so explicitly if handed to
  statutory reporting.

## Verification

```bash
./odoo-bin -c <conf> -d <db> -u ayadi_consolidation_report --test-enable --test-tags /ayadi_consolidation_report --stop-after-init
```
Then in the UI: Accounting → Reporting → Consolidation Report → pick the two companies
sharing a coded account → confirm one merged line appears, and that re-running the
wizard while a *different* company is active in the switcher gives the same total in
the same target currency.

## References

- `addons/account/models/account_account.py` — `code` field SQL (`_field_to_sql`),
  `group_id`/`_compute_account_group`, `AccountGroup.company_id` (Odoo 19)
- Related: [[multicompany-consolidated-report-currency-follows-active-company]]
- Project file: `addons/ayadi_consolidation_report/wizard/consolidation_report_wizard.py`

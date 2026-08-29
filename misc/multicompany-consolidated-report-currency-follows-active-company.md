# Multi-Company Consolidated Report (P&L/Balance Sheet) Shows a Different Total Depending on Which Company Is "Current"

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | misc (accounting / account_reports)        |
| Odoo Versions | 19 (mechanism present since the CTA currency-table rewrite; verify on 17/18) |
| Severity      | 🔴 Critical |
| Last Verified | 2026-08-29 |
| Author        | ENG/Gamal Mansour |

**Tags:** `account_reports`, `consolidation`, `multi-company`, `multi-currency`, `currency_translation`, `cta`, `profit-and-loss`, `res.currency`

---

## Problem

User has 4 companies, each with its **own currency** (EGP, AED, EUR, USD) and each maintaining
its **own `res.currency.rate` table** (rates are company-specific in Odoo, confirmed via
`SELECT currency_id, company_id, name, rate FROM res_currency_rate` — the same currency pair
has different rate rows under different `company_id`s).

When viewing a consolidated report (e.g. **Profit and Loss**) with all 4 companies allowed/checked
in the company switcher, the report shows a different total depending on which company is set as
the **current/active** company (the star, not the checkboxes) — even though the underlying journal
entries did not change.

## Root Cause

This is **not a bug in custom code** — it is core `account_reports` behavior, and it is easy to
mistake for a bug because nothing in the UI tells the user the reporting currency silently
changed.

`account.report` multi-company/multi-currency reports (any report with `currency_translation`
set, e.g. P&L has `currency_translation = 'cta'`) convert every company's balances through a
temporary `account_currency_table`, built by:

```
addons/account/models/res_currency.py :: ResCurrency._create_currency_table()
    main_company = self.env.company   # <-- the CURRENTLY ACTIVE company, not a fixed "group" company
    ...
    main_company_unit_factor = main_company.currency_id._get_rates(main_company, date_to)[...]
```

So:
1. The **target currency** of the whole consolidation is always `self.env.company.currency_id`
   — i.e. whichever company is currently selected as "active" in the company switcher (the
   radio/star), **not** the set of companies checked as visible/allowed.
2. The **rates used to convert every other company's balances into that target currency** are
   looked up from `self.env.company`'s **own** `res.currency.rate` table (`_get_rates` is called
   on `main_company`).

Consequence: switching the *active* company between A (EGP) and C (EUR) doesn't just change the
display currency of the same number — it changes **both** the target currency **and** which
company's rate table is used to do every cross-currency conversion in the consolidation. Since
each company's rate table is entered independently (real-world FX rates from different sources/
dates), the two totals are not simply a currency conversion of each other — they can be
genuinely inconsistent.

For `currency_translation = 'cta'` reports specifically, it's even more nuanced: equity accounts
use the `'historical'` rate, income/expense (P&L) accounts use the `'average'` rate for the
period, and balance-sheet accounts use `'current'` — all three rate sets are (re)computed from
whichever company is currently active (see `_get_table_builder_historical/_average/_current` in
the same file).

### Project-specific compounding factor

This project also has a custom module, `ayadi_multicurrency_reports`
(`addons/ayadi_multicurrency_reports/models/account_report.py`), that adds a "display in any
currency" dropdown on top of the core report. Its `_get_lines()` override applies **one flat
spot rate** (`company_currency._get_conversion_rate(company_currency, target_currency,
self.env.company, report_date)`) on top of whatever core already computed, and its default
currency (`_init_options_display_currency`) is `self.env.company.currency_id` — i.e. it **also**
re-derives its base currency from whichever company is currently active. Two behaviors follow:

- If the user never touches the currency dropdown, `is_base` stays `True` and the module is a
  no-op — it does **not** protect the user from the core behavior described above.
- If the user *did* previously pick an explicit display currency (persisted via
  `previous_options`) and then switches the active company, the module now performs a **second**
  conversion on top of core's already-converted total, again sourcing the rate from whichever
  company is now active — compounding the inconsistency rather than fixing it.

## Solution ✅

There is no single-line code fix — this is a design choice in core Odoo. Two complementary
answers, pick based on what the client actually needs:

1. **Operational fix (no code, correct today):** always keep the **same one company** set as the
   *active/current* company (star) when reviewing a consolidated report; only toggle which
   companies are *checked* (included). The consolidated total will then always be expressed in
   that one company's currency, computed with that one company's rate table, and will be
   internally consistent run to run. Document this for the accountants — the checkboxes control
   *scope*, the star controls *reporting currency + rate source*.

2. **Data fix (if numbers still look "wrong" even from the same active company):** audit
   `res.currency.rate` across the 4 companies for the same currency pair/date — if the client
   wants a single source of truth for FX rates instead of one table per company, rates should be
   entered/updated on one company only (or synced), not re-keyed independently per company.

3. **If the client actually wants "always report in currency X regardless of which company is
   active"** (which is what `ayadi_multicurrency_reports` was trying to provide): the module
   needs to stop deriving its base/default currency from `self.env.company.currency_id` and
   instead default to (and, more importantly, **always run its conversion pass against**) a
   fixed configured "group reporting currency" — and importantly it should convert from **each
   company's own native balance** using a **single, fixed rate source**, not re-convert the
   already-core-converted (and already active-company-dependent) total. Concretely this means
   overriding `_init_options_currency_table`/forcing `self.env.company` for the duration of
   `_get_lines` to the fixed reporting company (via `with_company()`), not post-multiplying the
   core output by a spot rate. This is a 🔴 change (touches the currency table / consolidation
   engine) — treat as its own scoped task with a blast-radius review, don't bolt it on.

## ⚠️ Pitfalls

- Don't assume "different number when switching company" is a bug report to silently patch —
  verify first whether the user changed the *active* company (radio/star) vs. just the *allowed*
  companies (checkboxes). These are two different Odoo concepts and the UI does not visually
  distinguish their effect on report currency.
- `res.currency.rate` is **company-specific** in Odoo by default (`company_id` on the rate
  record). Don't assume one currency has one rate — always check `company_id` when debugging FX
  discrepancies.
- A custom "pick any display currency" report module that keys its default off
  `self.env.company.currency_id` will silently re-derive a new base every time the active company
  changes — it looks like it "remembers" a currency but it's actually re-defaulting, unless
  `previous_options` already carries an explicit prior selection.

## Verification

```sql
-- Compare rate tables for the same currency pair across companies — mismatches here
-- explain non-proportional totals between "active company" choices.
SELECT currency_id, company_id, name, rate
FROM res_currency_rate
WHERE currency_id IN (<ids of the currencies in play>)
ORDER BY currency_id, company_id, name DESC;
```
Then, in the UI: open the P&L with all companies checked, note the total and the active company;
switch only the active company (star) without touching the checkboxes; confirm the total changes
and that it corresponds to `previous_total / rate(new_active_company table)` — if it doesn't even
match that, the rate tables themselves are inconsistent (data issue, not just "expected
multi-currency conversion").

## References

- `addons/account/models/res_currency.py :: _create_currency_table` (Odoo 19 core)
- `enterprise/account_reports/models/account_report.py :: _init_options_currency_table`,
  `_currency_table_aml_join` (Odoo 19)
- Project file: `addons/ayadi_multicurrency_reports/models/account_report.py`

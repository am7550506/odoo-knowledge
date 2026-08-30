# Where to xpath-inject custom columns into the Journal Entry "Journal Items" tab (Odoo 19)

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | views                                      |
| Odoo Versions | 17, 18, 19                                 |
| Severity      | 🟢 Low                                     |
| Last Verified | 2026-08-30                                 |
| Author        | ENG/Mohamed Saber                          |

**Tags:** `account.move`, `account.move.line`, `view_move_form`, `xpath`, `journal items`

---

## Problem

You need to add a custom field (a Many2one, a computed field, etc.) as a column
on the editable Journal Items grid shown inside a Journal Entry / Invoice /
Bill form (`account.move` form view), and to the mobile fallback form for the
same lines — without touching `addons/account` core files.

## Root Cause

`account.view_move_form` embeds the lines twice inside `<page id="aml_tab" ...>`:
once as an editable `<list>` (desktop) and once as a `<form>` (mobile,
`mode="list,kanban"` on the `line_ids` field switches to the form on small
screens). Both blocks repeat several of the same field names (e.g.
`analytic_distribution`), so a naive single xpath on
`//field[@name='analytic_distribution']` matches twice and Odoo refuses to
apply the inherited view (ambiguous match).

## Solution ✅

Scope the xpath to the `<list>` and `<form>` separately, anchored on the
`aml_tab` page id (stable across versions since ~17.0):

```xml
<record id="view_move_form_my_module" model="ir.ui.view">
    <field name="name">account.move.form.my.module</field>
    <field name="model">account.move</field>
    <field name="inherit_id" ref="account.view_move_form"/>
    <field name="arch" type="xml">
        <xpath expr="//page[@id='aml_tab']//list//field[@name='analytic_distribution']" position="after">
            <field name="my_custom_field_id" optional="show"
                   domain="[('company_id', 'in', (False, company_id))]"/>
        </xpath>
        <xpath expr="//page[@id='aml_tab']//form//field[@name='analytic_distribution']" position="after">
            <field name="my_custom_field_id"/>
        </xpath>
    </field>
</record>
```

For the standalone "Journal Items" screen (Accounting > Reporting > Journal
Items) and its search, inherit these two instead (also stable ids):

- `account.view_move_line_tree` (list) — anchor xpath on
  `//field[@name='account_id']` (position `after`), it appears once.
- `account.view_account_move_line_filter` (search) — same anchor field
  `account_id` for adding a filter field; group-by filters can be injected
  with `<xpath expr="//search" position="inside"><group id="..."> ... `.

## ⚠️ Pitfalls

- A bare `//field[@name='analytic_distribution']` (no `//list` / `//form`
  scoping) matches both occurrences and Odoo raises a "view inheritance"
  error about the xpath not resolving to exactly one node — always scope by
  ancestor when a field name repeats across the list/form pair.
- If the custom field lives on a company-scoped model, add a `domain`
  filtering by `company_id` on the `account.move.line` row itself (it already
  has a `company_id` field) — the same pattern core uses for `account_id`
  (`domain="[('company_ids', 'parent_of', company_id)]"`). Don't reference
  `parent.company_id` here; inside the `line_ids` list/form the field context
  is the line, not the move.
- Don't confuse `view_account_move_line_filter` (id used at priority 16,
  the "Search Journal Items" view actually shown in the UI) with any other
  `account.move.line` search view of the same string in the same file —
  check the `priority` field and confirm with a targeted `awk` range dump
  before trusting a grep match.

## Verification

Install the module, open any Journal Entry, go to *Journal Items* tab —
new column appears next to Analytic; resize to a mobile viewport (or use the
Kanban/mobile form) — new field appears there too. Open *Accounting >
Reporting > Journal Items* — column and search/group-by are present.

## References

- Related file: `addons/account/views/account_move_views.xml` (search for
  `id="aml_tab"`, `view_move_line_tree`, `view_account_move_line_filter`).

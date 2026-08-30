# Search view `<group>` (Group By section) rejects `string`, `expand`, `id` — module install fails with a generic ParseError

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | views                                      |
| Odoo Versions | 17, 18, 19                                 |
| Severity      | 🔴 Critical                                |
| Last Verified | 2026-08-30                                 |
| Author        | ENG/Mohamed Saber                          |

**Tags:** `search view`, `group by`, `xml`, `RelaxNG`, `view_validation`, `install error`

---

## Problem

Installing/upgrading a module fails with a generic error that gives no real clue
about which attribute is wrong:

```
odoo.tools.convert.ParseError: while parsing .../my_views.xml:NN
Invalid view <model>.search definition in my_module/views/my_views.xml
```

This happens when adding the classic "Group By" section to a `<search>` view
using the pattern that was common in older Odoo (≤16):

```xml
<!-- ❌ fails on 17/18/19 -->
<group expand="0" string="Group By">
    <filter string="Company" name="group_by_company" context="{'group_by': 'company_id'}"/>
</group>

<!-- ❌ also fails: id is not a valid attribute on group either -->
<group id="groupby" string="Group By">
    ...
</group>
```

## Root Cause

The `ParseError` message is generic on purpose — the real reason is only logged
a few lines earlier, as `WARNING ... odoo.tools.view_validation` entries with
RelaxNG (`RELAXNGV`) errors, e.g.:

```
ERROR:RELAXNGV:RELAXNG_ERR_INVALIDATTR: Invalid attribute expand for element group
ERROR:RELAXNGV:RELAXNG_ERR_NOELEM: Expecting an element field, got nothing
ERROR:RELAXNGV:RELAXNG_ERR_EXTRACONTENT: Element search has extra content: field
```

Checking `odoo/addons/base/rng/common.rng`, the `<rng:define name="group">`
pattern only allows these attributes: `colspan`, `rowspan`, `fill`, `height`,
`width`, `name`, `color`, `invisible` (plus `groups` via `access_rights` and
`position` via `overload`). **There is no `string`, `expand`, or `id`
attribute on `<group>` at all.** Its children are `field*` followed by a
`container` (which does allow `filter`, so filters inside a bare `<group>`
are fine — only the attributes on `group` itself are the problem).

Core Odoo 17-19 search views never label or collapse the "Group By" section
via attributes on `<group>` — the frontend renders that block automatically
once it detects a `<group>` made only of `<filter>` elements with
`context="{'group_by': ...}"`. Confirmed against
`addons/account/views/account_move_views.xml`, which repeats this bare
pattern dozens of times:

```xml
<separator/>
<group>
    <filter string="Journal Entry" name="group_by_move" domain="[]" context="{'group_by': 'move_name'}"/>
    <filter string="Account" name="group_by_account" domain="[]" context="{'group_by': 'account_id'}"/>
    ...
</group>
```

## Solution ✅

Drop `string=` and `expand=` entirely, and use `name=` (not `id=`) if you
need to target the group later via xpath:

```xml
<separator/>
<group name="groupby_my_module">
    <filter string="Company" name="group_by_company" context="{'group_by': 'company_id'}"/>
</group>
```

## ⚠️ Pitfalls

- The `ParseError` traceback alone will not show you the RelaxNG detail —
  you must grep the **server log** (or console) for `odoo.tools.view_validation`
  / `RELAXNGV` lines emitted a few lines *before* the `ParseError`, at the
  same timestamp. The hint in the error ("Restart with
  `--log-handler odoo.tools.convert:DEBUG`") gives a fuller Python traceback
  but still won't show the RNG-level attribute name — the `view_validation`
  WARNING lines already logged are the fastest path to the real cause.
- Don't add `id=` to `<group>`, `<page>` in your OWN new views expecting it
  to work like `<record id="...">` — most view-arch elements use `name=` as
  their addressable identifier; `id=` on an arch element is only valid on
  the small set of elements whose RNG definition explicitly lists it (e.g.
  `<page id="...">` does allow it in `account.view_move_form` — always check
  the specific element's RNG pattern rather than assuming `id` is universal).
- This is not new in 19 — the same schema shape is already true on 17 and
  18; a codebase inherited from ≤16 with `<group expand="0" string="...">`
  patterns will break the moment it's touched on 17+.

## Verification

```bash
./odoo-bin -c <conf> -d <db> -u <module> --stop-after-init
```
Then grep the log for `RELAXNGV` — a clean install/upgrade produces none for
this module's views. Open the search view's filter dropdown in the UI and
confirm the Group By section renders with the expected filters.

## References

- `odoo/addons/base/rng/common.rng` (`<rng:define name="group">` and
  `<rng:define name="container">`)
- Related file: [account-move-line-journal-items-list-xpath-odoo19.md](account-move-line-journal-items-list-xpath-odoo19.md)

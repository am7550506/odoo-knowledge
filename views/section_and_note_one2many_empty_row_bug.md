# section_and_note_one2many Widget Deletes Empty Row / Missing Name Field

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | views                                      |
| Odoo Versions | 16, 17, 18, 19                             |
| Severity      | 🔴 Critical                                |
| Last Verified | 2026-05-31                                 |
| Author        | ENG/Gamal Mansour                          |

**Tags:** `views`, `one2many`, `section_and_note_one2many`, `ui`, `tree`, `required`

---

## Problem

When using `widget="section_and_note_one2many"` on an O2M field, clicking "Add a section" or "Add a note" creates an empty row, but there is no text box to type the name. Clicking on the row immediately deletes it. If you try to drag and drop (reorder) the section without typing a name, you get an RPC validation error like: `"The following fields are invalid: [Your O2M Field Name]"`.

## Root Cause

1. **Hardcoded Field Name in JS:** The `SectionAndNoteListRenderer` in Odoo core heavily hardcodes checking for `col.name === "name"`. If your Python model uses a field named `description` instead of `name` for the section text, the Javascript widget will filter it out, rendering a completely empty row.
2. **Missing Inline Tree View:** The JS component often fails to parse the DOM columns correctly if the `<tree>` view is defined externally. It heavily prefers the `<tree>` view to be defined **inline** inside the `<field name="o2m_field">` tag to properly hook into the architecture.
3. **Missing `display_type` in Form View:** If you add `required="not display_type"` on any field in a list/form view, `display_type` must be present in that view (e.g., `<sheet><field name="display_type" invisible="1"/></sheet>`), otherwise Odoo throws a `ParseError` during module upgrade.
4. **`required=True` in Python Model:** If the `name` (Description) field is strictly `required=True` in Python, Odoo will block saving an empty section (e.g., if the user creates a section and drags it before typing a name).

## Solution ✅

**1. Rename `description` to `name`:**
Your model MUST use a field exactly named `name` for the section title:
```python
name = fields.Text(string='Description', tracking=True)
```

**2. Make `name` Optional in Python:**
Remove `required=True` from the `name` field in the Python model to allow users to drag/drop empty sections without validation errors. You can optionally enforce it in the XML instead.

**3. Inline the Tree View:**
Move the tree view directly inside the O2M field declaration in your parent form view:
```xml
<field name="boq_line_ids" widget="section_and_note_one2many" mode="tree">
    <tree editable="bottom">
        <control>
            <create name="add_line_control" string="Add a line"/>
            <create name="add_section_control" string="Add a section" context="{'default_display_type': 'line_section'}"/>
            <create name="add_note_control" string="Add a note" context="{'default_display_type': 'line_note'}"/>
        </control>
        <field name="display_type" column_invisible="True"/>
        <field name="sequence" widget="handle"/>
        <!-- Your specific fields -->
        <field name="name" widget="section_and_note_text"/>
    </tree>
</field>
```

**4. Include `display_type` in Form View:**
If your O2M model has a Form view, ensure you include `<field name="display_type" invisible="1"/>` inside the `<sheet>` to prevent XML ParseErrors on modifiers.

**5. Conditional Required for other fields:**
For other fields (e.g., `boq_type`), remove `required=True` from Python, and use `<field name="boq_type" required="not display_type"/>` in the XML.

## ⚠️ Pitfalls

- **Do NOT use `is_section` boolean:** Do not use a boolean field for section filtering or domains. Use the built-in `display_type = fields.Selection([('line_section', 'Section'), ('line_note', 'Note')])`. When passing `default_display_type` in the `<create>` context, Odoo DOES NOT automatically update custom boolean fields like `is_section`. Rely purely on `display_type`.
- **Smart Button Counts:** If your parent model has a compute field counting the O2M records (e.g. `len(rec.line_ids)`), it will incorrectly count the section and note rows too. You MUST filter them out:
  ```python
  rec.line_count = len(rec.line_ids.filtered(lambda l: not l.display_type))
  ```
- **Action Domains:** If you have standalone menu actions or smart buttons opening a separate list view of the lines, the sections and notes will show up as empty/broken rows. You MUST add `('display_type', '=', False)` to the `<field name="domain">` of these `ir.actions.act_window` actions.

## Verification

1. Click "Add a section" - a new row should appear spanning all columns with a text input.
2. Type a name and drag it - it should save successfully without "Invalid fields" errors.
3. Try dragging it *before* typing a name - it should also drag seamlessly if `name` is no longer strictly required in Python.

# QWeb Report Table Zebra Striping and Colspan Layout Alignment

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Category      | views                                      |
| Odoo Versions | 15, 16, 17, 18, 19                         |
| Severity      | 🟡 Medium                                  |
| Last Verified | 2026-05-20                                 |
| Author        | ENG/Mohamed Saber                          |

**Tags:** `qweb`, `report`, `pdf`, `bootstrap`, `zebra-striping`, `colspan`, `table`

---

## Problem

When customizing QWeb reports, the developer may encounter two visual/layout issues:
1. The table contains automatic alternating gray/white (zebra) striping on body rows that is inherited from Odoo's core stylesheets (usually via bootstrap classes or core report SCSS), which conflicts with custom designs where normal rows should remain plain and only section/note rows should have a background color.
2. In custom tables, certain columns (like `Amount` or total columns) seem to fall "outside the table" visually. This happens because special rows like sections, notes, or subtotals have an incorrect `colspan` that is less than the total columns of the table. As a result, the background colors and bottom/top borders of these rows stop before the last column, leaving a blank vertical strip on the right side of the table.

## Root Cause

1. Zebra striping is applied globally by Odoo's standard `.o_main_table` styles or standard `.table` bootstrap stylesheets.
2. The `colspan` variable in `<thead>` is often calculated dynamically to exclude the `Amount` or `Price` column because developers copy default templates where `colspan` was defined only up to the second-to-last column. When rendering rows like section, note, or subtotal using `<td t-att-colspan="colspan">`, the `colspan` is exactly `total_columns - 1`, which means the last column is completely ignored in that row's layout, causing borders and cell backgrounds to terminate early.

## Solution ✅

### 1. Remove Zebra Striping & Style Section Rows

Incorporate a scoped CSS `<style>` block inside the report's `page` template to clear row and cell background colors, and apply a premium background color (e.g., Tailwind Slate Slate-100 or soft gray) to section rows dynamically using conditional classes:

```xml
<div class="page">
    <style>
        /* Remove zebra-striping from report tables */
        .o_main_table tbody tr {
            background-color: transparent !important;
        }
        .o_main_table tbody tr td {
            background-color: transparent !important;
        }
        /* Custom clean slate/gray styling for section rows to look extremely premium */
        .o_main_table tbody tr.o_line_section {
            background-color: #f1f5f9 !important; /* Premium Tailwind Slate-100 */
            font-weight: bold;
        }
        .o_main_table tbody tr.o_line_section td {
            background-color: #f1f5f9 !important;
        }
    </style>
    
    ...
    
    <tbody class="sale_tbody">
        <t t-foreach="doc.breakdown_analysis_ids" t-as="line">
            <tr t-att-class="'bg-light fw-bold o_line_section' if line.display_type == 'line_section' else ('font-italic o_line_note' if line.display_type == 'line_note' else '')">
                ...
            </tr>
        </t>
    </tbody>
</div>
```

### 2. Fix Colspan to Span All Columns

Ensure `colspan` is initialized to the **total number of columns** in the table (including the `Amount` column), and dynamically incremented if dynamic columns (such as Discount) are present:

```xml
<thead>
    <tr>
        <!-- Set colspan to total columns initially (e.g. 5: Description, Qty, UoM, Unit Price, Amount) -->
        <t t-set="colspan" t-value="5" />
        <th class="text-left">Description</th>
        <th class="text-right">Quantity</th>
        <th class="text-right">Measurement</th>
        <th class="text-right">Unit Price</th>
        <th t-if="display_discount" class="text-right" groups="sale.group_discount_per_so_line">
            <span>Disc.(%)</span>
            <t t-set="colspan" t-value="colspan+1" />
        </th>
        <th class="text-right">Amount</th>
    </tr>
</thead>
```

Ensure all section, note, and subtotal rows use `t-att-colspan="colspan"`:

```xml
<t t-if="line.display_type == 'line_section'">
    <td t-att-colspan="colspan">
        <b t-field="line.name" />
    </td>
</t>
```

## ⚠️ Pitfalls

- **Do not use hardcoded `colspan="99"`:** While it might make a cell span the whole width, it is non-standard and can cause layout/border rendering quirks in older versions of `wkhtmltopdf`.
- **CSS specificity:** Always use `!important` inside the `<style>` block when overriding bootstrap report classes because `wkhtmltopdf` has high specificity rules for `.table` and `.table-striped`.
- **Dynamic Columns:** Always count columns carefully. If you add or remove columns (like taxes or discounts), ensure the `colspan` matches the exact number of header cells.
- **Bootstrap 5 Text Alignment Compatibility (Odoo 18+):** In newer Odoo versions using Bootstrap 5, the traditional `text-right` utility class is deprecated and has no effect. Always use `text-end` on the `tr`/`td` cells. For absolute robustness inside wkhtmltopdf reports, combine it with an inline style on the cell, e.g., `<td class="text-end" style="text-align: right !important;">` to ensure that subtotals or amounts align correctly to the right margin.

## Verification

Deploy the change, print the PDF report, and verify:
1. Normal rows have a clean, solid background (white/transparent) with no zebra striping.
2. Section rows are highlighted in a subtle, beautiful Slate-100 or soft gray background.
3. The table's right border and backgrounds span perfectly all the way to the right side of the `Amount` column, and nothing feels "outside" the table.

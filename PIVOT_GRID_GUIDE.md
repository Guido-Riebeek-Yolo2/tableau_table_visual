# Pivot Grid: team guide

Pivot Grid is a Tableau viz extension that replaces the native text table. Authors build it like a normal sheet. Viewers can rearrange it like an Excel pivot table, sort it, add totals, copy values and export exactly what they see to Excel.

This guide covers:

1. [Adding the extension to a sheet](#1-adding-the-extension-to-a-sheet)
2. [The Marks card tiles](#2-the-marks-card-tiles)
3. [What viewers can do](#3-what-viewers-can-do)
4. [What authors can do](#4-what-authors-can-do)
5. [The Format Extension panel](#5-the-format-extension-panel)
6. [Parameter-driven fields](#6-parameter-driven-fields)
7. [How totals are calculated](#7-how-totals-are-calculated)
8. [Scenarios](#8-scenarios)
9. [What is saved, where, and for whom](#9-what-is-saved-where-and-for-whom)
10. [Limitations](#10-limitations)
11. [Troubleshooting](#11-troubleshooting)

---

## 1. Adding the extension to a sheet

Requirements: Tableau 2024.2 or later (Desktop, Server or Cloud). On Server/Cloud the extension URL must be safe-listed by an admin:
`https://guido-riebeek-yolo2.github.io/tableau_table_visual/index.html`

1. Create a new worksheet.
2. On the Marks card, open the mark type dropdown and choose **Add Extension**.
3. Choose **Access Local Viz Extensions** and select `pivot-grid.trex`.
4. Allow full data access when Tableau asks.
5. Drag fields onto the extension's tiles on the Marks card (see section 2).

Notes:

- Tableau reads the `.trex` file when the extension is added. If the `.trex` file ever changes, the extension has to be removed and added again. Changes to the extension itself (the hosted `index.html`) reach every workbook automatically, without re-adding.
- Removing and re-adding the extension resets all its settings on that sheet. Export the settings first (Format panel › Import / export).

---

## 2. The Marks card tiles

The extension adds four tiles to the Marks card. They set the starting layout; viewers can change it afterwards.

| Tile | Accepts | Max | What it does |
|---|---|---|---|
| **Rows** | Dimensions | 10 | Fields shown as row headers, outer to inner in tile order. |
| **Columns** | Dimensions | 10 | Fields shown as column headers, outer to inner in tile order. |
| **Values** | Measures | 30 | Measures shown in the grid, in tile order. |
| **Fields** | Dimensions and measures | 30 | Fields that are available but not on the grid at first. They appear in the viewer's field list, ready to drag onto the grid. They also hold helper fields used only for totals (section 7). |

### What each tile costs

Everything on the Marks card is part of the sheet's level of detail. Tableau queries the data at the combination of every dimension on every tile, and sends that table to the extension.

- **Dimensions add rows.** A dimension on the Fields tile multiplies the data by its number of values, even when it isn't on the grid. Only put dimensions there that viewers really need, and avoid high-cardinality fields (players, transactions).
- **Measures add columns.** They're cheap. Extra measures on the Fields tile (for example as ratio helpers) barely affect performance.

### Measure Names

The grid has a **Measure Names** pseudo-field, like Tableau. It starts as the outermost field on Columns. Viewers (and the author, while editing) can drag it to any position on rows or columns.

### Field names

- Measures show without their aggregation: `SUM(Bet Eur)` shows as "Bet Eur".
- Date parts show as Tableau does: `YEAR(Calendar Date)` shows as "Year of Calendar Date".
- Fields driven by a parameter show the parameter's current value as their name (section 6).

---

## 3. What viewers can do

Everything in this section is available to everyone who opens the view. Viewer changes last only while the view is open. The next visit starts from the author's layout again.

### Toolbar

| Button | What it does |
|---|---|
| **Arrow (Layout)** | Hides or shows the Columns, Rows and Values shelves. |
| **Reset layout** | Appears when the viewer's grid differs from the author's layout. Goes back to the author's layout. |
| **Sort grid (n)** | Opens the multi-level sort menu. The number shows how many sort levels are active. |
| **Group rows** | On: repeated row labels merge into one label per group. Off: every row repeats its full label. |
| **Expand rows** | Shows one row per group with a + to open it, like an accordion. Needs Group rows on and two or more row fields. |
| **Totals** | Turns on grand totals and subtotals, sets each total's position independently, and chooses which metrics are totalled. |
| **.00→.0 / .0→.00** | Shows fewer or more decimals for all measures, from 6 fewer to 6 more. The badge shows the shift (+2, −1). |
| **Copy n cells** | Appears while value cells are selected. Copies them. |
| **Export to Excel** | Downloads the grid exactly as shown. |

### Rearranging the grid

- **Drag** any field pill between the Columns and Rows shelves, or directly onto the grid's column header area or row header area. An orange marker shows where it will land.
- **Drag a field to the field list** (left side) to take it off the grid. Its values are then combined (see section 7).
- **Drag a field from the field list** onto Rows, Columns, or the grid.
- **Drag a measure** within the Values shelf to reorder it, to the field list to remove it, or from the field list onto Values or anywhere on the grid to add it.
- **Click a measure pill** on the Values shelf to hide it (it's struck through). Click again to show it. This is never saved, not even for the author.
- **Fold the field list** with its arrow. Dropping a field on the folded strip still removes it from the grid.
- The field list is sorted A to Z, dimensions first, then measures.

Keyboard, on a focused pill:

| Key | Effect |
|---|---|
| Left / Right | Move the pill within its shelf |
| Up | Rows to Columns; from the field list, add to Columns (or add a measure to Values) |
| Down / Enter | Columns to Rows; from the field list, add to Rows |
| Delete / Backspace | Remove to the field list |
| Space / Enter on a measure | Hide or show it |
| Escape | Cancel a drag |

### Sorting

- **Header buttons.** Every value column header and every field name in the grid's header corner has a sort button.
  - Value column: 1st click largest first, 2nd click smallest first, 3rd click removes the sort.
  - Field: 1st click A to Z, 2nd click Z to A, 3rd click removes the sort.
  - A button you click becomes the first sort level. With several levels, the buttons show their position (1, 2, ...).
- **Sort grid menu.** Lists the sort levels in order ("Sort by", "Then by"...). Each level is a row field, a column field, or the values in one column, with a direction. Levels can be added, reordered, removed or cleared.
- **Group rows on:** each level is sorted within its parent. Sorting by values sorts the groups by their subtotal, then the rows inside each group.
  - For a measure with no correct subtotal (rule "Leave blank" or "From LOD measures"), groups keep their natural order and only the rows inside them are sorted.
- **Group rows off:** all rows are sorted as one flat list, so rows of the same group can end up far apart. Subtotals are hidden in this mode (a notice says so).

### Totals

Open **Totals** in the toolbar:

- **Grand total** for rows and/or columns.
- **Subtotals per [field]** for every field that has another field inside it. Subtotals follow the field if it's moved to the other axis.
- Each grand total and subtotal has its own position selector: **Top / Bottom** for rows, **Left / Right** for columns. For example, row grand totals can be at the top while column grand totals are on the right. The default is top/left; existing saved layouts keep their previous positions.
- **Metrics included** defaults to **All metrics**. Clear it and tick only the metrics you want to total. This applies to both grand totals and subtotals, without hiding any metric's ordinary data cells or collapsed-group values.

When Measure Names is on the total's axis, only selected metrics get total rows or columns. When it is on the other axis, unselected metrics' total cells stay blank. Selecting no metrics removes all total rows and columns.

To sort by one metric's overall total, enable **Columns > Grand total**, include only that metric, then use the sort button on its grand-total column or choose that column in **Sort grid**. With Measure Names on rows, that total column uses the selected metric to sort groups; with multiple metrics selected, it uses the first selected metric in the Values order.

How the numbers are calculated depends on each measure's total rule (section 7). If a measure can't be totalled correctly, its total cells stay blank, never wrong.

### Expand rows

With **Expand rows** on (and Group rows on, and 2+ row fields):

- Each row field that has another field inside it gets +/− boxes. A closed group shows one row with that group's totals.
- The row field's header has Expand all / Collapse all. Expand all opens that level and those above it; collapse all closes it and those below.
- Which groups are open is part of the author's layout. Viewers' changes last for the session.

### Tooltips

Hovering a value cell shows each row field and column field with its value (or Total / Grand total), then the measure and its value.

### Selecting and copying cells

- **Click** a value cell to select it; click it again to deselect.
- **Ctrl/Cmd+click** adds or removes cells.
- **Shift+click** selects the rectangle from the last clicked cell. Ctrl+Shift+click adds that rectangle.
- **Esc**, or clicking a header, clears the selection.
- Copy with **Ctrl/Cmd+C** or the **Copy n cells** button. Values are copied as displayed, tab-separated, ready to paste in Excel. Only the selected rows and columns are included; unselected cells in between stay empty. Headers aren't copied.
- **Tableau Desktop:** Desktop handles Ctrl+C itself and may copy the whole sheet. Use the **Copy** button there. On Server/Cloud, Ctrl+C works.

### Export to Excel

- Exports exactly what's on screen: layout, sorting, totals, hidden measures, expanded/collapsed groups, number formats and decimals.
- The first rows hold the sheet name and the export date and time.
- Numbers are real Excel numbers with a matching number format, so they can be summed in Excel.
- Grouped row labels are merged cells when Group rows is on, repeated when it's off.
- Colours and fonts aren't exported.
- File name: `<sheet name>_<YYYY-MM-DD>.xlsx`.

---

## 4. What authors can do

An author is anyone editing the sheet: Tableau Desktop, or web editing on Server/Cloud.

### The layout saves itself

While editing, every change to the grid is saved automatically as the layout everyone opens with:

- field order on rows and columns, and the position of Measure Names
- measure order on Values, and which measures are removed to the field list
- sorting, totals, Group rows, Expand rows and which groups are open
- the decimals shift
- whether the shelves and the field list are collapsed

**Save the workbook (and publish) to keep it.** Struck-through measures (clicked to hide) are never saved.

To start over from the Marks card arrangement: Format panel › **Layout** › **Back to Marks card arrangement**.

If the author later adds, removes or moves fields on the Marks card, the saved layout adjusts: new fields go to their tile's starting place, removed fields drop out, and fields moved to another tile follow the tile.

### The Format Extension panel

On the Marks card, click **Format Extension**. Tableau only shows this button to people editing the sheet; viewers can't open the panel.

- Changes preview live.
- **Save** stores the settings in the workbook. Save and publish the workbook to keep them.
- **Cancel** (or Esc) undoes everything since the panel was opened.
- **Reset to defaults** resets **all** settings, including number formats, total calculations and parameter names. Use Cancel if that was a mistake.

---

## 5. The Format Extension panel

One section is open at a time. Each collapsed section shows a short summary of its settings.

| Section | What it controls |
|---|---|
| **Font** | Font family, size (px) and text colour for the whole grid. |
| **Field names** | Colour and bold/italic/underline of field name labels in the header corner. |
| **Column headers** | Background, text colour, style and alignment of column headers. |
| **Row labels** | Background, text colour, style and alignment of row labels. |
| **Cells** | Background, style and alignment of value cells; row banding on/off and band colour. |
| **Totals** | Background, text colour and style, separately for subtotals and grand totals. |
| **Lines and spacing** | Grid line colour, group divider colour, and spacing (compact, normal, roomy). |
| **Number formats** | Per measure, and per parameter value: Tableau's format or a custom one (section 5.1). |
| **Date formats** | Per true date field: Tableau's format or a fixed pattern (section 5.2). |
| **Total calculations** | Per measure, and per parameter value: how totals and combined cells are calculated (section 7). Also the **Used for totals only** list. |
| **Parameter names** | Which parameter names each Grid Level / Metric Selector field (section 6). |
| **Layout** | When the layout was last saved; **Back to Marks card arrangement**. |
| **Import / export** | Copy settings to other sheets (section 5.3). |

### 5.1 Number formats

1. Choose the **Measure**.
2. If the measure is driven by a parameter, choose what the format **Applies to**: "All values (default)" or one parameter value (section 8.4).
3. **Format**:
   - **Automatic (Tableau)**: the format set in Tableau. This is the default.
   - **Number**: decimal places, display units (K, M, B), prefix (e.g. €), suffix, thousands separator, negative style (-1234 or (1234)).
   - **Percentage**: the value is treated as a fraction, so 0.123 shows as 12.3%.
   - For a single parameter value, "Same as all values" uses the "All values" setting.
4. The example line shows a real value from the sheet in the chosen format.

Custom formats apply everywhere: cells, totals, tooltips, copy and Excel export. The toolbar's decimals buttons shift them further.

Measures marked "• custom" in the Measure list have their own format (for all values or for at least one parameter value).

### 5.2 Date formats

Only for fields Tableau sends as real dates (for example an exact date). Date parts like YEAR() or WEEKDAY(), and dates that come from a calculation as text (such as Grid Level calcs), keep their own format.

Formats are display only. A format coarser than the data, such as "Q1 2026" on daily dates, shows repeated labels; it doesn't combine the days. Use the right date level in Tableau to combine them.

### 5.3 Import / export

Reuse settings on other sheets:

1. Tick the parts to copy:
   - **Fonts, colours and text styles**
   - **Number and date formats**
   - **Total calculations**, including per-value rules and the totals-only list
   - **Parameter names** (off by default, because these are usually sheet specific)
2. **Copy settings** (clipboard) or **Download file** (`pivot-grid-format_<sheet>.json`).
3. On the other sheet, open the panel, tick the same parts, then paste into the Import box and click **Import pasted**, or use **Load file…**.
4. Click **Save**.

Number formats, date formats and total rules are stored by field name. They only take effect on sheets with fields of the same name. The import message says how many settings don't match a field on this sheet.

Use this as a backup before removing an extension from a sheet.

---

## 6. Parameter-driven fields

Many of our sheets use parameters to pick which field is shown, for example `Grid Level 1` driven by `Kp.03 Grid Level 1 (P)`, or `Kp.06 Metric Selector` driven by `Kp.06 Metric Selector (P)`.

### The field shows the parameter's value as its name

A field whose name contains **Grid Level** or **Metric Selector** shows the current value of its parameter as its name everywhere: headers, shelves, field list, menus, Excel export. When the parameter changes, the name changes straight away.

How the parameter is found: Tableau doesn't tell extensions which parameter a calculation uses, so the extension matches by name. The parameter is the one whose name contains the field name as whole words (so `Grid Level 1` doesn't match `Grid Level 10`). If several match, the shortest name wins.

If that's wrong, or if several parameters match, use Format panel › **Parameter names**. Each Grid Level / Metric Selector field has a dropdown: Automatic, a specific parameter, or **Don't rename**. If several parameters match and none is chosen, the grid shows a notice naming the field.

### "None" hides the field

When the parameter's value is "None", "(None)", "- None -", "-", "n/a" or empty, the field is left out of the grid, the shelves, the field list, the totals and the export. It keeps its position, so it comes back in the same place when the parameter changes again.

### Metric selector calculations

How the selector calculation is written matters:

- **Aggregated form (recommended):**
  ```
  CASE [Kp.06 Metric Selector (P)]
  WHEN 6 THEN SUM([Bet Eur])
  WHEN 7 THEN SUM([Bet Eur]) / SUM([Bet Qty])
  ...
  END
  ```
  The field is `AGG(Kp.06 Metric Selector)`. Tableau calculates every value correctly, including ratios. The extension only needs to know how to total each metric (section 8.4).
- **Row-level form:**
  ```
  CASE [Kp.06 Metric Selector (P)]
  WHEN 6 THEN [Bet Eur]
  WHEN 7 THEN [Bet Eur] / [Bet Qty]
  ...
  END
  ```
  The field is `SUM(Kp.06 Metric Selector)`. This works for metrics that can be added up. **Ratios are wrong in Tableau itself**: Tableau divides on every data row and then adds the results up. The extension can't fix that.

**Rule:** if any metric in the selector is a ratio or average, write the whole calculation in the aggregated form. Tableau doesn't allow a CASE that mixes the two forms.

Changing a calculation from one form to the other changes the field's name (`SUM(...)` vs `AGG(...)`). Settings saved for the old name no longer apply and have to be set again.

---

## 7. How totals are calculated

Values on the grid come straight from Tableau when a cell matches one row of Tableau's data. The extension calculates a value itself when a cell covers several rows of data:

- **subtotals and grand totals**
- **closed groups** with Expand rows
- **cells after a dimension is moved to the field list**, because each cell then covers all of that dimension's values

Each measure has a **total rule** that says how to combine those rows. If the rule can't give the right answer, the cell stays blank and a notice names the measure. The extension never shows a total it knows may be wrong.

The adjacent-period difference rules are an exception: they recalculate every displayed cell from a source measure, including deepest-level cells, instead of using Tableau's precomputed table calculation.

### Automatic rules

Without a setting, the rule comes from the field's aggregation:

| Aggregation | Automatic rule |
|---|---|
| SUM, COUNT, CNT | Sum |
| MIN | Minimum |
| MAX | Maximum |
| CNTD / COUNTD of a field that is also on the sheet | Count distinct of that field |
| AVG, MEDIAN, ATTR, AGG (calculations), CNTD of a field not on the sheet, others | Blank |

### Rules the author can choose

Format panel › **Total calculations** › choose the **Measure** › **Totals use**:

| Rule | Calculation | Use for |
|---|---|---|
| **Automatic** | As in the table above. | Plain sums, MIN, MAX. |
| **Sum** | Adds the values. | Additive measures, including AGG calculations that are additive. |
| **Minimum / Maximum** | Smallest / largest value. | MIN/MAX measures. |
| **Ratio of two measures** | Sum of the numerator ÷ sum of the denominator over the rows covered. | Averages and ratios: Avg Bet = Bet Eur ÷ Bet Qty, Margin = GGR Eur ÷ Bet Eur, Hold %. |
| **Difference from adjacent period** | Source at the current member minus source at the member selected by the offset, aggregated at the same displayed level. | Month-to-month Bet Eur or GGR Eur differences, including collapsed rows and totals. |
| **Percentage difference from adjacent period** | Difference divided by the absolute comparison value. Blank when that value is zero. | Percentage changes recalculated at each displayed level, not added from child rows. |
| **Count distinct of a field** | The number of distinct values of a chosen field among the rows covered (where the measure isn't zero). | Distinct counts such as number of operators. |
| **From LOD measures** | Takes the value from a helper LOD measure that Tableau calculated for that total level. | Non-additive measures when Count distinct isn't practical. |
| **Leave blank** | Always blank. | Measures that should never be totalled. |

Measures can't use themselves as numerator or denominator, and both must be on the sheet (section 7.1).

### 7.1 Helper measures and "Used for totals only"

Ratio, Count distinct and LOD rules need extra fields in Tableau's data:

- **Ratio:** the numerator and denominator measures (e.g. Bet Eur, Bet Qty).
- **Adjacent-period differences:** the source measure (e.g. Bet Eur). It can also remain visible on Values. Its calculation must be Sum, Minimum, Maximum, Ratio or Count distinct, not another period comparison or an LOD/blank rule.
- **Count distinct:** the counted dimension (e.g. Operator).
- **LOD:** the LOD helper measures.

Put them on the **Fields** tile.

- Measures on the Fields tile can be ticked under Format panel › Total calculations › **Used for totals only**. Ticked measures never show on the grid or in the viewer's field list.
- LOD helper measures are hidden from the field list automatically.
- A counted dimension on the Fields tile is a normal field: viewers can drag it onto the grid.

---

## 8. Scenarios

### 8.1 Additive measures (Bet Eur, GGR Eur, Round Qty)

`SUM(...)` measures total correctly with the automatic rule. Nothing to set up.

### 8.2 A fixed ratio or average measure (Margin, Avg Bet)

The measure is a calculation like `SUM([GGR Eur]) / SUM([Bet Eur])`, so the field is `AGG(Margin)` and its totals are blank by default.

1. Put **GGR Eur** and **Bet Eur** on the Fields tile.
2. Format panel › Total calculations › Measure **Margin** › Totals use **Ratio of two measures**.
3. Numerator **GGR Eur**, Denominator **Bet Eur**.
4. Optional: tick GGR Eur and Bet Eur under **Used for totals only** if viewers shouldn't see them.
5. Save.

Every total and combined cell is now total GGR ÷ total Bet, which is the correct margin. An average of margins or a sum of margins would be wrong.

Don't use `AVG([x])` for averages that should be weighted. Totals of an AVG measure are blank unless a rule is set, and the right rule is a ratio of the underlying sums.

### 8.3 Distinct counts (number of operators, players)

**Option A: Count distinct of a field (preferred for small fields)**

1. Put the counted field (e.g. **Operator**) on the Fields tile.
2. If the measure is `CNTD(Operator)`, this works automatically. For a calculation (e.g. `AGG(Operator Qty)` = `COUNTD([Operator])`), choose **Count distinct of a field** and pick **Operator**.

This is correct at every level and in every arrangement. The cost is more data: one row per operator per grid row. That suits fields with hundreds or thousands of values, not millions (players).

With the counted field in the data, other non-additive measures on the sheet need a rule too (for example a ratio), or their cells go blank.

**Option B: From LOD measures (for large fields such as players)**

1. For each total level, create a helper measure in Tableau that excludes the fields being totalled, for example:
   - Subtotal per Grid Level 1: `MIN({EXCLUDE [Grid Level 2] : COUNTD([Player])})`
   - Grand total: `MIN({EXCLUDE [Grid Level 1], [Grid Level 2] : COUNTD([Player])})`
2. Put the helper measures on the Fields tile.
3. Total calculations › choose the measure › **From LOD measures**. Add each helper and tick the fields it excludes.

Use MIN, MAX or ATTR around the LOD, not SUM (SUM multiplies it by the number of rows). Totals without a matching helper stay blank, for example after a viewer moves fields around.

**Note on "actives" metrics:** Daily Actives and similar are distinct counts of players. Summing them across operators or days counts a player once per operator or per day. Sum is fine if that's the intended meaning (e.g. "active days"). If the total must be unique players, use Option B.

### 8.4 A metric selector parameter with mixed metrics (Kp.06)

Example: Kp.06 offers First Time Actives, Daily Actives, ..., Bet Eur, **Avg Bet Eur**, GGR Eur, ... and maybe **Margin**. Most are sums; Avg Bet Eur and Margin are ratios. The grid shows one metric at a time.

Total rules and number formats can be set **per parameter value**. Values without their own setting use the "All values" setting.

1. Write the selector calculation in the aggregated form (section 6).
2. Put the ratio inputs on the Fields tile: **Bet Eur**, **Bet Qty**, and **GGR Eur** for Margin.
3. Format panel › Total calculations › tick them under **Used for totals only**.
4. Total calculations › Measure: the selector. It's listed under the parameter's current value, e.g. "Daily Game Actives".
   - **Applies to: All values (default)** › Totals use **Sum**.
   - **Applies to: Avg Bet Eur** › **Ratio of two measures** › Bet Eur ÷ Bet Qty.
   - **Applies to: Margin** › **Ratio of two measures** › GGR Eur ÷ Bet Eur.
5. Number formats › Measure: the selector.
   - **Applies to: All values (default)** › Number, 0 decimals.
   - **Applies to: Avg Bet Eur** › Number, 1 decimal, prefix €. The settings start as a copy of "All values", so only the differences need changing.
   - **Applies to: Margin** › Percentage, 1 decimal.
6. Save, then save and publish the workbook.

In the **Applies to** list, values with their own setting are marked "• custom" and the parameter's current value is marked "(current)". When the viewer changes the parameter, totals and formats switch straight away.

Things to know:

- The list of values comes from the parameter, so new values (such as Margin) show up automatically.
- Settings are stored under the parameter's **value** (e.g. 7), not its display name. Renaming "Avg Bet Eur" keeps the settings. **Reordering or renumbering the parameter's values moves the settings to other metrics**, so check them after changing the list.
- This only works for parameters with a list of values. For a range or free-text parameter, only the current value is offered.
- Another way, needing no per-value setup: the **From LOD measures** rule with helpers such as `MIN({EXCLUDE [Grid Level 1] : [Kp.06 Metric Selector]})`. Tableau then calculates each total correctly for every metric. It needs one helper per total level, and totals without a matching helper stay blank.

### 8.5 A viewer moves a dimension to the field list

Example: rows are Operator › Brand, and the viewer drags Brand to the field list. Each Operator row now covers all its brands. The values are combined with the same total rules as subtotals:

- Sum measures: added up.
- Ratios with a Ratio rule: recalculated correctly.
- Measures with no rule (AVG, COUNTD without its field, AGG calculations): blank, with a notice. Set a total rule to fix this.

Moving a field between Rows and Columns never needs combining. Ordinary values come straight from Tableau; adjacent-period difference rules continue to calculate from their source measure.

### 8.6 Parameter-driven grid levels

Example: Rows has Grid Level 1, Grid Level 2 and Grid Level 3, each driven by a parameter.

- Each level shows the parameter's value as its name, e.g. "Operator", "Brand".
- A level set to "None" disappears from the grid and comes back in the same place.
- Subtotals per level follow the level, not the parameter value.
- LOD helpers that exclude `[Grid Level 1]` etc. keep working whichever field the parameter picks.

### 8.7 Same formatting on many sheets

Set up one sheet, then Format panel › Import / export › tick Fonts and Number formats (and Total calculations if the measures have the same names) › Copy settings. Paste into the Import box on each other sheet and Save.

### 8.8 Different decimals for one view

Use the toolbar's decimals buttons. The author's choice is saved with the layout; a viewer's choice lasts for the session. For permanent per-measure decimals use Number formats.

### 8.9 Month-to-month differences at every row level

Example Tableau calculation: `ZN(SUM([Bet Eur])) - LOOKUP(ZN(SUM([Bet Eur])), 1)`. Tableau calculates it at the worksheet's grain and does not know when a viewer collapses Brand into Operator inside the extension.

1. Keep **Bet Eur** and the output measure **Bet Eur Diff** on the sheet. Bet Eur can be visible on Values or hidden on Fields; the output may be your existing table calculation or an appropriately named numeric placeholder calculation.
2. Open Format panel > **Total calculations** > Measure **Bet Eur Diff**.
3. Set **Totals use** to **Difference from adjacent period**.
4. Set **Source measure** to **Bet Eur**, **Across** to **Calendar Month**, and **Offset** to **1**.
5. Save the format, then save and publish the workbook.

For **Bet Eur Diff%**, use **Percentage difference from adjacent period** with the same source, field and offset. Set its Number format to **Percentage** if Tableau's existing formatting is not already a percentage. Repeat for GGR Eur using **GGR Eur** as the source.

Offsets count members in the comparison field's ascending or descending field-sort order, not raw date intervals. With months newest-first, +1 compares October against September; -1 compares against the preceding displayed member. Sorting rows by values does not change this comparison order. Parent fields before the comparison field on its axis define partitions: Year > Month restarts the comparison within each year.

The extension aggregates the source at the current row level for both periods, then subtracts. Missing/null source values for an existing comparison member are zero, so a missing Brand month does not skip to an older month when Calendar Month is on columns. A field member absent from the entire partition is not available for comparison; this does not create missing calendar months or retrieve dates filtered out of Tableau.

No adjacent member, a zero percentage denominator, an unsupported source, or a comparison field removed from the grid produces a blank. A total that aggregates away Calendar Month is also blank: there is no individual month to compare. Row grand totals that retain a specific Calendar Month are recalculated correctly. These rules apply equally to deepest rows, collapsed groups, subtotals, sorting, tooltips, copy and Excel export, and support per-parameter-value settings and format import/export.

---

## 9. What is saved, where, and for whom

| What | Who changes it | Saved | Viewers |
|---|---|---|---|
| Grid layout (rows, columns, measure order, sorting, totals, Group rows, Expand rows, open groups, decimals, collapsed shelves and field list) | Author, while editing | Automatically, in the workbook. Save and publish the workbook. | Start from it on every visit. Their own changes last only while the view is open. **Reset layout** returns to it. |
| Hidden (struck-through) measures | Anyone | Never | Session only |
| Format panel settings | Author | On **Save**, in the workbook. Save and publish the workbook. | Always see them; can't change them. |
| Parameter values | Anyone | As Tableau normally saves parameters | As in Tableau |

All settings belong to the extension instance on that sheet. Removing and re-adding the extension starts from scratch: export the settings first.

---

## 10. Limitations

- **Data volume.** Tableau sends all the sheet's data to the browser. Very large grids (tens of thousands of cells or more) get slow. At about 400 pages of data Tableau stops sending; the grid then shows "Only the first N rows were loaded". Add filters.
- **Fields tile dimensions** increase the data (section 2).
- **Subscriptions.** Excel subscriptions contain Tableau's own export of the sheet's data (every field on the Marks card, unpivoted, without the extension's totals or formatting). Image and PDF subscriptions don't show viz extensions. Use the extension's Export to Excel, or a separate crosstab sheet for subscriptions.
- **Excel export** has no colours or fonts.
- **Totals** can only be calculated from the data on the sheet. Tableau doesn't give extensions calculation formulas or a way to query another level of detail, so non-additive measures need a total rule.
- **Date formats** only apply to true date fields.
- **Tableau Desktop Ctrl+C** may copy the whole sheet; use the Copy button.
- **Per-value settings** are tied to the parameter's values (section 8.4).

---

## 11. Troubleshooting

| Symptom | Cause and fix |
|---|---|
| Totals of a measure are blank, with a notice "... can't be added up" | The measure has no correct automatic rule (AVG, AGG calculation, COUNTD). Set a rule under Total calculations (section 7). |
| Totals of an average or ratio look far too high | The rule is Sum, which adds up averages. Use **Ratio of two measures** (sections 8.2 and 8.4). |
| A metric selector's ratio is wrong even in normal cells | The calculation is in the row-level form. Rewrite it in the aggregated form (section 6). |
| Settings disappeared after changing a calculation | The field's name changed (e.g. `SUM(...)` to `AGG(...)`). Set them again or import them with the new name. |
| Per-value settings apply to the wrong metric | The parameter's values were reordered or renumbered. Check each value under **Applies to**. |
| "Subtotals are hidden while rows are sorted without grouping" | Turn on **Group rows**, or remove the sort. |
| "Expand rows works on grouped rows" | Turn on **Group rows**. Expand rows also needs two or more row fields. |
| "... match several parameters" | Choose the right parameter under Format panel › **Parameter names**. |
| A field shows its own name instead of the parameter value | Its name doesn't contain "Grid Level" or "Metric Selector", or no parameter name contains it. Check under **Parameter names**. |
| "Some fields could not be read from the sheet" | Open the view with `?debug=1` (local preview) to see which fields and columns Tableau sent. |
| "Only the first N rows were loaded" | Too much data. Add filters or remove dimensions from the Fields tile. |
| Format Extension button missing | It's only shown while editing the sheet. |
| Layout changes aren't kept for viewers | Save and publish the workbook after arranging the grid. |
| Grid shows "Couldn't load the Tableau library" or "Add this as a viz extension" | The extension was added as a dashboard extension, or the hosted files are incomplete. Add it on the Marks card of a worksheet. |

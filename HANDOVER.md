# Handover: Tableau Pivot Grid viz extension

This file brings a new session up to speed on the project: what it is, what's built, how the code works, what's unverified, and what to build next. Read it fully before changing code.

## 1. The problem and the goal

Tableau's native text tables can't be rearranged by viewers, and exporting them doesn't put the current parameter values into the headers. The team previously solved the export half with a **dashboard extension** ("Grid Export", the original `index.html` + `tableau-export.trex` in the project). It reads a worksheet's summary data and writes an Excel file whose headers come from parameter display aliases.

The new goal is a **viz extension** (worksheet extension, Tableau 2024.2+) that replaces the native text table:

- The author builds it like a normal sheet. Fields go onto custom **Rows**, **Columns** and **Values** tiles on the Marks card.
- The viewer can drag fields between rows and columns, like an Excel pivot table but better.
- A sort button on every column and measure. *(Next iteration.)*
- Subtotals and grand totals that the viewer turns on when they want them. *(Next iteration.)*
- An Excel and CSV export with parameter values in the titles. *(Next iteration.)*
- Saving a custom view restores the grid exactly as the viewer left it. *(Later iteration.)*

Decisions already made with the user:

- Data volume is a known limitation and accepted for now. All data comes into the browser.
- Deployment is fine. The extension is hosted on GitHub Pages and Server/Cloud admins can safe-list it.
- Moving a field between rows and columns does **not** need re-aggregation. The set of cells stays the same and only the layout changes, so values always come straight from Tableau. Re-aggregation is only needed for subtotals, grand totals, or removing a dimension from the grid.

## 2. Files

```
index.html                          The whole extension: HTML, CSS and JS in one file
pivot-grid.trex                     Viz extension manifest
tableau.extensions.1.latest.min.js  Extensions API library, version 1.17 (from tableau/extensions-api)
xlsx.full.min.js                    SheetJS 0.20.3 for Excel export (loaded on first export)
fonts/                              Noto Sans woff2 files (latin, latin-ext, italic) + OFL.txt licence
HANDOVER.md                         This file
```

Repo: `https://github.com/Guido-Riebeek-Yolo2/tableau_table_visual`, served by GitHub Pages from `main` / root. Demo data only appears in a local preview (`file://` or localhost). Hosted, a missing library or failed connection shows an error instead.

Reference files from the original export module, which live in the project root and not in `pivot-grid/`:

- `index.html` is the old Grid Export dashboard extension. Its parameter header logic will be reused for the export.
- `tableau-export.trex` is the old manifest.
- `tableau_extensions_1_latest_min.js` is the Extensions API library, version 1.17. It contains `getVisualSpecificationAsync`, `worksheetContent`, `getSummaryDataReaderAsync` and `SummaryDataChanged`. The page loads it as `./tableau.extensions.1.latest.min.js`, so rename the file when you copy it in.
- `xlsx_full_min.js` is SheetJS, for the export iteration. The old module loads it as `./xlsx.full.min.js`.

Hosting URL in the manifest: `https://guido-riebeek-yolo2.github.io/tableau_table_visual/index.html`

## 3. Conventions to keep

- **Generic by design.** The same file must run unmodified on every worksheet and dashboard. Never hardcode report names, field names or measure lists. Settings go in the `CONFIG` block at the top of the script.
- **Single file**, plain JS, no build step and no frameworks. Libraries are loaded from the same folder.
- **Numbers come from Tableau.** Show `formattedValue` from the summary data, read with `applyWorksheetFormatting: true`. Never re-format numbers that Tableau already formatted.
- **Never show a wrong total.** If a total can't be calculated correctly, leave the cell blank. See section 8.2.
- UI copy is short and in sentence case. Errors say what happened and how to fix it.
- The visual language follows Tableau: blue `#4a97b8` for dimension pills, green `#1aa874` for measures, orange `#e8762d` for the drop marker, and Benton Sans with system fallbacks.

## 4. How `index.html` works

### 4.1 Data model (`buildModel`)

The model is built from the summary data, independently of any layout:

```
model = {
  dims:     [{ id, label, col, axis: "rows"|"cols"|"fields", members: [labels in sort order], index: Map(label -> memberIdx) }],
  measures: [{ id, label, col, rule: "sum"|"min"|"max"|"none", fmt: number -> string }],
  records:  [{ k: [memberIdx per dim, in model.dims order], m: [cell per measure] }],   // every summary row
  dimPos:   Map(dimId -> position in model.dims),
  measById: Map(measId -> { ...measure, index }),
  dupes:    count of rows whose full dim key repeats (e.g. fields on the Detail tile)
}
```

- `rule` comes from the aggregation in the field name through `COMBINE_RULE` (SUM/COUNT sum, MIN, MAX). Anything else is `none`.
- `fmt` is learned from Tableau's own formatted cells by `inferFormat`: prefix/suffix, separators, decimals, %, K/M/B units and negative style. Only used for combined cells.

- A cell is `{ f: formatted text, s: sort key }`. The sort key is `nativeValue` when it's a number, `getTime()` for a Date, and the formatted text otherwise.
- Members are sorted by sort key. Numbers sort numerically. Text uses `Intl.Collator` with `numeric: true`, which matches Tableau's natural sort ("1x2Network, 3Oaks, 7rings, 100HP Gaming").
- Field ids are the Tableau field names, such as `SUM(Bet Eur)` or `YEAR(Calendar Date)`. Display labels come from `stripAgg` for measures and `prettyDim` for dimensions (`YEAR(x)` becomes "Year of x").

### 4.2 Layout

```
layout = { rows: [ids], cols: [ids], measures: [measure ids, display order], hidden: [measure ids], removed: [dim ids] }
```

- Dims not on `rows` or `cols`, and measures not in `measures`, show in the field list on the left. `removed` remembers dims and measures the viewer took off the grid, so a refresh doesn't put them back.
- Fields on the author's **Fields** tile start in the field list. That tile accepts dimensions and measures; a name with an aggregation (`SUM(x)`, `CNTD(x)`...) is treated as a measure.
- The field list folds to a 28px strip with its arrow button. Dropping on the strip still removes a field.

- `MEAS = "__measures__"` is the Measure Names pseudo-field. It can sit at any position on either axis.
- `defaultLayout()` builds the author's layout from the encodings. It places `MEAS` according to `DEFAULT_MEASURES_AXIS` and `DEFAULT_MEASURES_AT`.
- `reconcile(old, prev)` runs on every data refresh. It keeps the viewer's arrangement, adds new fields to their default place, drops fields that no longer exist, and re-applies the author's choice for any field the author moved to another tile.
- Every layout change goes through `commit(next, movedId)`. It skips no-op changes, re-renders, and briefly flashes the moved pill. The callers are `moveField`, `moveMeasure`, `toggleMeasure`, Swap and Reset.

### 4.3 Pivot (`axisTuples`, `firstDiff`, `runLens`)

- `axisTuples(levels, vis)` returns one tuple per header path. Each tuple holds member indices aligned with the levels. At the `MEAS` level it holds an index into `vis`, the visible measures. Only dimension combinations that exist in the data are produced, as in Tableau. Tuples are sorted lexicographically.
- `firstDiff(T)[i]` is the level at which tuple `i` first differs from tuple `i-1`.
- `runLens(fd, lvl)` gives the span length at each run start and 0 elsewhere. It drives `colspan` and `rowspan`.
- To read a cell value, `gridView(rowLv, colLv)` groups the records by the dims on the grid. A group of one record keeps Tableau's cells as they are. A larger group (a dim is in the field list, or Detail fields split the data) is combined with `combine` using each measure's `rule`; `none` gives a blank cell and a notice naming the measure. The key is built from `view.ids` with `where`.

### 4.4 Render (`render`)

The table is built as one HTML string:

- **thead** has one `tr.hr` per column level. Each starts with a corner cell `th.clabel` holding the level's field name as a draggable `.ftag`, followed by `th.ch` header cells with colspans. Outer headers that span several columns wrap their label in `.stick`, so it stays in view when scrolling sideways.
- After those comes one `tr.lrow`, the row-label row. It holds a `th.rlabel` per row level and a `th.lfill` filler cell.
- **tbody** has row headers `th.rh` with rowspans, then `td` value cells.
- If an axis has no levels, a placeholder cell reading "Drop a field here" acts as a drop target.
- `measureSticky()` measures the header row heights and row-header column widths after rendering. It sets the CSS variables `--t{n}`, `--l{n}` and `--lsum` on `#grid`, which the inline `style="top:var(--tN)"` and `left:var(--lN)` attributes use.
- Group boundaries: `tr.gb` gets a stronger top border and `.gc` a stronger left edge (inset box-shadow).

### 4.5 Drag and drop

- The drag engine uses pointer events, not HTML5 drag-and-drop, so it behaves the same in Tableau Desktop's browser, on Server/Cloud and on touch screens.
- Any element with `data-drag` (the field id) and `data-kind` can be dragged. `dim` and `meas` items go to rows or cols; `dim` items can also go to the field list (`avail` zone). `measure` items reorder within the Values shelf, go to the field list, or (from the list) drop anywhere on the grid to join Values.
- A drag starts after 4px of movement. Without that movement it counts as a click, and clicking a measure pill toggles it hidden.
- `zones(kind)` is recomputed on every pointer move, so scrolling mid-drag works. The zones are:
  - the shelves, with pills laid out as `dir: "wrap"`;
  - a grid column zone covering the header area above `tr.lrow`, with `dir: "v"` because levels stack vertically;
  - a grid row zone covering the row-header area, with `dir: "h"`.
- `updateTarget` finds the insertion point as a `beforeId`, not a numeric index. That keeps it correct even when the grid hides the `MEAS` level. It also positions the orange `#marker`.
- Escape cancels a drag.
- Keyboard support on a focused pill: Left and Right reorder it, Up moves a field from Rows to Columns, Down moves it from Columns to Rows, Delete removes it to the field list, and Space or Enter toggles a measure. In the field list, Up adds to Columns and Down or Enter adds to Rows.

### 4.6 Tableau integration

- `init()` uses demo data only when running locally (`file://` or localhost) and `?demo=1` is set, the library is missing, or `initializeAsync` takes longer than `INIT_TIMEOUT_S`. When hosted it never shows demo data: it shows "Still connecting to Tableau…" after the timeout but keeps waiting, and shows an error if the library is missing or initialisation fails. If `worksheetContent` is missing, the page shows "Add this as a viz extension", because it was loaded as a dashboard extension.
- `loadFromTableau()` works in this order:
  1. Calls `getVisualSpecificationAsync()`, takes `marksSpecifications[activeMarksSpecificationIndex].encodings`, and maps each `e.id` to an axis through `ENC_AXIS`.
  2. Reads the summary data with `getSummaryDataReaderAsync(undefined, { ignoreSelection: true, applyWorksheetFormatting: true })` and `getAllPagesAsync()`, always calling `releaseAsync()` afterwards.
  3. Matches each encoding field name to a column with `matchColumn`: an exact match first, then whitespace- and case-normalised, then with the aggregation stripped.
- `refreshFromTableau()` is tied to `SummaryDataChanged` and coalesces bursts of events into one load at a time.
- `?debug=1` shows a diagnostics panel with the encodings, the summary columns with their data types, unmatched fields, model stats and the layout JSON.

## 5. Unverified against real Tableau (check these first)

None of this has run inside Tableau yet. Demo mode and a mock worksheet shaped like the real API have been tested.

Verified against Tableau's docs and samples: encodings are `{ id, field: { name } }` and the ConnectedScatterplot sample indexes summary data by `columns[].fieldName === encoding.field.name`. A role-spec can list several role types. At most four custom encodings are allowed, and we use all four. `getAllPagesAsync` stops at 400 pages and pads the array with empty slots; the code filters those out and warns.

1. **Encoding field name vs. summary column name.** Expected to match exactly per the sample. Check with `?debug=1`, which also logs the raw `encodings[].field` objects.
2. **Encoding structure.** We assume several fields on one tile appear as several entries with the same `id`, in tile order. Verify, especially that the order matches the tile.
3. **Manifest.** Check that the `role-type` values `discrete-dimension` and `continuous-measure`, `<fields max-count>`, and `min-api-version 1.11` are accepted. Compare with Tableau's official viz extension samples, such as the Sankey sample in the `tableau/extensions-api` GitHub repo.
4. **Init in Desktop.** On a slow Server the "Still connecting" hint may appear before data loads; it clears itself once Tableau responds.
5. **Summary data order and nulls.** Check how Null members and aliases come through. Members are keyed by `formattedValue`.
6. **Whether `applyWorksheetFormatting: true` behaves as expected for viz extensions.**

## 6. Testing

Open `index.html` from disk. The Tableau library will be missing, so the page runs on demo data: about 28 suppliers × Product Type × 2 years × 4 measures.

The page was tested with Python Playwright and headless Chromium (binary at `/opt/pw-browsers/chromium-1194/chrome-linux/chrome` in the sandbox it was built in). Simulate drags with `page.mouse.move`, `page.mouse.down`, about 10 intermediate moves, then `page.mouse.up`, because a single jump won't pass the 4px threshold the way a real drag does. Check the result with `page.evaluate("JSON.stringify(layout)")`. You can also drive the layout directly with `commit({...layout, rows:[...], cols:[...]})`.

Cases already covered:

- drags from the shelves and from the grid headers
- Measure Names onto rows
- every field on one axis, and all fields on rows
- hiding measures, including all of them
- Swap, Reset, Hide and Show layout
- the keyboard moves
- sideways scrolling with sticky outer labels

## 7. Known limitations of the current build

- No virtualization. Every cell is rendered as DOM, which is fine up to tens of thousands of cells. Add row virtualization before large sheets.
- Outer row headers use rowspan and stick left. Very tall groups keep their label at the top of the group, not sticky vertically.
- Viewers can remove a dimension from the grid. Measures whose aggregation isn't SUM, COUNT, MIN or MAX show blank in merged cells until the author-configured rules from section 8.2 exist.
- The layout lives only in memory and is lost on reload (section 8.4).
- There is no dark-mode styling, which is fine inside Tableau.

## 8. Roadmap

### 8.0 Current batch of requests (from the user, in order)

Status: `[ ]` open, `[x]` done. Answers to clarification questions are recorded under each item.

1. [x] **Full-height drop indicator for rows.** When dragging a field from columns to rows in the grid, the orange marker must run down the whole table, not just the row-label header cell, so it's obvious the field is going to rows.
2. [x] **Collapse arrow for the shelves.** A small arrow, like the one on the Fields list, to hide the Columns, Rows and Values shelves.
   - Answer: one arrow collapses all three shelves together; it replaces the Hide button.
3. [x] **Row grouping toggle.** On: repeated row values are grouped (one merged label per group, as now). Off: every row shows its full label, repeated on each row.
   - Answer: a "Group rows" toolbar toggle, on by default.
4. [x] **Subtotals and grand totals.** The viewer can switch them on and choose the level(s) at which subtotals and grand totals appear.
   - Answer: a "Totals" toolbar menu with a checkbox per row and column field, plus "Grand total" for rows and for columns.
   - Answer: totals for both rows and columns.
   - Answer: default position is the top of each group and the first row / first column. The user can switch to bottom / right.
5. [x] **Export to Excel.**
   - Answer: exactly what's on screen (layout, totals, sort, hidden measures), with title rows: sheet name, export date and current parameter values. Numbers as real Excel numbers; grouped rows merged when grouping is on, repeated when off.
6. [x] **Sort any column by its values.** Dimension columns (e.g. Supplier) sort A–Z; measure columns sort largest to smallest.
   - Row grouping **on**: sort groups by their subtotal, then the items within each group.
   - Row grouping **off**: sort all rows as one flat list, so one supplier's games can end up far apart (game 1 in row 1, game 3 in row 32, games 4–5 in rows 60–61).
   - Answer: 1st click = default direction (measures largest first, dimensions A–Z), 2nd click = reverse, 3rd click = unsorted.
   - Answer: for measures with no correct subtotal (AVG, COUNTD), groups keep their normal order and only the items inside each group are sorted.

**How the batch was built** (all in `index.html`):

- Toolbar (always visible): shelf collapse arrow (`#btnShelves`, toggles `#app.collapsed`), Sort grid menu (`#sortMenu`), Group rows, Totals menu (`#totalsMenu`), Export to Excel, Format (authors).
- `layout` gained `grouped`, `totals: { sub: [field ids], grandRows, grandCols, atEnd }` and `sort: { type: "dim", id, dir } | { type: "val", sig, dir }`. Subtotals are stored per field id, so they follow a field to the other axis. `sig` identifies a column by its members and measure, independent of level order.
- `cellAt(fixed, measureIndex)` computes a value at any level of detail (cells, subtotals, grand totals) from the records, cached per set of fixed dims, using the measure's combine rule.
- `axisItems(levels, vis, opt)` builds the ordered header paths of an axis, including total paths (`t[i] = -1` means "all members", `tot` = totalled level, -1 for grand). Grouped order sorts each level within its parent; ungrouped value or dimension sorts use `flatCmp` over all rows. Subtotal rows can't sit inside groups that a flat sort has split up, so they're hidden with a notice.
- `buildGrid()` returns everything `render()` and `exportExcel()` need, so the export matches the screen exactly.
- Excel: SheetJS (`xlsx.full.min.js`, 0.20.3, lazily loaded). Numbers are native values with a number format derived from Tableau's formatting (`inferFormat(...).xl`). Title rows hold the sheet name and export time. The file is named `<sheet>_<YYYY-MM-DD>.xlsx`.
- Still to confirm inside Tableau: downloads from a viz extension frame (Desktop and Cloud), and the parameter list.

7. [x] **Parameter values as field names.** A field on the grid driven by a parameter (`Grid Level 1` <- `Kp.03 Grid Level 1 (P)`) shows the parameter's current value ("Operator") in headers, shelves, the field list, the Totals menu and the export.
   - Matching: only fields whose name matches `PARAM_FIELDS` (`/grid level|metric selector/i`). The parameter is the one whose name contains the field name as whole words, so `Grid Level 1` doesn't match `Grid Level 10`; the shortest match wins. Works for measures too (`SUM(Kp.06 Metric Selector)`).
   - Labels update on `ParameterChanged` without waiting for the data reload. `?debug=1` lists the field-to-parameter matches under PARAMETER LABELS.
   - `getParametersAsync()` returns **every parameter in the workbook**, and the API doesn't expose calculation formulas, so the extension can't see which parameter a field's calc actually uses. Matching is by name; the shortest matching name wins by default (a global `Grid Level 1 (P)` beats `Kp.03 Grid Level 1 (P)`). The Format panel's **Parameter names** section has one dropdown per Grid Level / Metric Selector field on the sheet: Automatic, any matching or other parameter, or "Don't rename". Choices are saved per field id in `format.paramMap`. With several candidates and no choice, the grid shows a notice naming the field.
8. [x] **Hide fields set to None.** When the parameter's value is in `PARAM_NONE` ("None", "(None)", "-", empty...), the field is marked `off` and left out of the grid, shelves, field list, totals and export. It keeps its place in `layout`, so it comes back in the same position when the parameter changes. Values stay exact: a None field holds one member, so leaving it out merges nothing.
9. [x] **Multi-level sort; Swap and Reset removed.** The toolbar's "Sort grid" menu lists sort levels in order ("Sort by", "Then by"...). Each level is a row field, a column field, or the values in a column, with a direction; levels can be added, reordered, removed or cleared.
   - `layout.sort` is an ordered array of keys `{ type: "dim", id, dir } | { type: "val", sig, dir }`. Earlier keys win; later ones break ties; natural order breaks the final tie.
   - Grouped rows: each level is sorted within its parent using the keys that apply to it (its own field's key; value keys by the group's subtotal). Ungrouped: all rows compared key by key as one flat list. Column fields' keys order the columns.
   - Header sort buttons edit the same list: an unsorted field becomes the first key, then reverses, then is removed. With several keys, buttons show the key's position.
10. [x] **Export without parameter list.** Title rows are only the sheet name and export date/time.
11. [x] **Format panel (authors only).** Opens only from the Marks card's **Format Extension** button (manifest `<context-menu>` + `initializeAsync({ configure })`), which Tableau shows only to people editing the sheet. Viewers can't open it. The toolbar Format button exists only in the local preview.
   - Settings: font family (Noto Sans first) / size / colour. Field names, column headers, row labels, cells, subtotals and grand totals each have colour settings plus Bold / Italic / Underline toggles; column headers and row labels have alignment. Also banding on/off and colour, cell alignment, spacing (compact/normal/roomy), grid line and group divider colours, and which parameter names each Grid Level / Metric Selector field.
   - Live preview while editing; Cancel/Escape reverts; Save writes JSON to `tableau.extensions.settings` key `format` (saved with the workbook). Values are validated on load (`cleanFormat`: hex colours, clamped size, allowlisted text).
   - Applied as `--g-*` CSS variables on the document; the grid CSS reads only those. Excel export doesn't carry colours (SheetJS community edition has no cell styles).
   - Layout: collapsible sections (`FORMAT_SECTIONS`, `<details>`), one open at a time: Font, Field names, Column headers, Row labels, Cells, Totals, Lines and spacing, Number formats, Date formats, Parameter names. Each collapsed section shows a live summary (colour swatches, style, counts) via `desc(format)` / `updateSummaries()`.
12. [x] **Value tooltips.** Hovering a value cell shows each row field then each column field in grid order (`Field: member`, or Total / Grand total), then `Measure: value`. A custom `#tip` element (instant, styled), positioned at the cursor and kept inside the window; hidden on scroll, drag and re-render. Built from `lastGrid`, so it always matches the cell.
13. [x] **Per-measure number formats and date formats** (Format panel).
   - Number formats: pick a measure, then Automatic (Tableau's own format, the default) / Number / Percentage, decimal places, display units (K/M/B), prefix, suffix, thousands separator, negative style. A live example uses a real value of that measure. Saved in `format.measureFormats[measureId]`; `makeNumberFormat` builds the formatter (workbook locale via `environment.locale`) plus the matching Excel format code (`.xl`), so exports match. Applied in `buildGrid`'s `cell()`, so grid, totals, tooltips and export all agree. Percentage assumes the value is a fraction (0.123 shows as 12.3%).
   - Date formats: only for true date fields (`nativeValue` is a Date, flagged `dt` in `cellOf`, `d.isDate` in the model). Patterns in `DATE_FORMATS`, formatted in UTC by `formatDate`; examples in the dropdown use the field's first date. Saved in `format.dateFormats[dimId]`. Display only: members stay distinct, so a format coarser than the data (e.g. Q1 2026 on daily dates) shows repeated labels — use a month/quarter date level in Tableau to aggregate. Date parts (YEAR()) and dates returned as text (Grid Level calcs) can't be reformatted.
14. [x] **Total calculations** (Format panel section; replaces the planned Configure dialog from 8.2). Per measure: Automatic (from the aggregation: SUM/COUNT sum, MIN, MAX, else blank), Sum, Minimum, Maximum, **Ratio** (numerator ÷ denominator), **From LOD measures**, or Leave blank. Saved in `format.totalRules[measureId]`; `applyFieldFormats` sets `m.rule`, `m.ratio = [numIdx, denIdx]`, `m.lods = [{ idx, ex: [dimPos] }]`.
   - Ratio: every combined cell = Σ numerator ÷ Σ denominator over the rows it covers (e.g. Margin = GGR ÷ Bet, Avg Bet = Bet Eur ÷ Bet Qty). Numerator and denominator must be summable and in the summary data (Values or Fields tile).
   - LOD (for COUNTD and other non-additive measures): the author adds helper measures like `MIN({EXCLUDE [Grid Level 1] : COUNTD([Operator])})` to the Fields tile and ticks which fields each excludes. For a total cell, `combine` takes the helper whose excluded fields equal the fields being totalled (ignoring single-member fields such as None grid levels), and only if its value is identical on every row covered; otherwise the cell stays blank. Helpers are hidden from the field list. Use MIN/MAX/ATTR around the LOD, not SUM (SUM multiplies it by the row count).
   - Sorting groups by a measure with rule `none` or `lod` keeps groups in natural order (LOD subtotals may not exist at every level).
   - **Count distinct** (`countd`, preferred over LOD): the counted dimension (e.g. Operator) is in the summary data, normally via the Fields tile; every combined cell = number of distinct members of that dimension among covered rows whose measure is non-zero. Correct at every level and arrangement, with no per-scenario setup. Auto-applied when the measure is named `CNTD(X)`/`COUNTD(X)` and dimension X is on the sheet; for calcs (e.g. `AGG(Operator Qty)` = COUNTD([Operator])) the author picks the counted field. Cost: the data grows by the counted field's cardinality per grid row, so it suits hundreds/thousands of values, not millions (players). With the counted field in the grain, other non-additive measures need a rule too (e.g. ratio), or their base cells go blank.
   - Why not automatic for everything: the Extensions API exposes neither formulas nor a way to query another level of detail; the extension only ever gets the summary table at the sheet's grain.
15. [x] **Import / export of format settings** (last Format panel section). Tick parts: fonts/colours/text styles (all non-keyed settings), number and date formats, total calculations, parameter names (off by default; sheet specific). Copy settings (clipboard, with a select-and-Ctrl+C fallback when Tableau's frame blocks the clipboard) or Download file (`pivot-grid-format_<sheet>.json`). Import by pasting or Load file….
   - Format: `{ pivotGridFormat: 1, style, measureFormats, dateFormats, totalRules, paramMap }`. Import merges the ticked parts into the current settings and runs `cleanFormat(next, format)`, so invalid values keep the current setting. Keyed settings (by measure/field id) only take effect where the sheet has the same fields; the status line says how many don't match. Imports preview immediately; Save persists, Cancel reverts.
16. [x] **Saved layouts** (options 1 and 2 from the persistence discussion).
   - **Author default**: in authoring mode (`environment.mode === "authoring"`: Desktop, or editing on Cloud; `?author=1` simulates it locally) every layout change, shelf collapse or field-list fold is saved automatically (debounced 600 ms) as `{ layout, ui: { shelvesCollapsed, sideFolded }, savedAt }` in workbook settings key `layout`, like arranging a normal sheet. The author must save and publish the workbook. Authors never use browser-stored layouts. Format panel › Layout shows the save time and offers "Back to Marks card arrangement". (First version needed a manual "Use current layout as default" click; the user's Desktop arrangement didn't reach Cloud because of that.)
   - **Viewer's own layout**: every layout change, shelf collapse or field-list fold is written to `localStorage` key `pivotgrid:view:<instanceId>` with `basedOn` = the default's `savedAt`. On load it's used only if `basedOn` matches the current default, so a newer author default replaces older personal layouts. A **Reset layout** toolbar button appears only when the viewer's layout differs from the default.
   - `instanceId` (settings) is created on the first author save so browser keys don't collide across workbooks; before that the key is sheet name + a hash of the field ids. All stored layouts pass through `cleanLayout` and then `reconcile`, so missing or new fields are handled.
   - `store` wraps settings: workbook settings once Tableau is connected (`tableauReady`), localStorage in the local preview so reload behaviour can be tested.
   - Measures struck through by a click (`layout.hidden`) are session-only: `persistable()` drops them from every saved layout (author and viewer) and from the Reset layout comparison.
   - Settings reset only when the extension is removed and re-added (a new instance); export/import formatting first. Option 3 (Custom Views via a hidden parameter) is still open in 8.4.
17. [x] **Expand rows (accordion).** Toolbar toggle `#btnExpand`, `layout.expand` (default off, saved with the layout like Group rows). Works only with Group rows on and 2+ row dimensions; otherwise a notice says so.
   - Every row dimension with another dimension after it gets a +/− box. A closed group is one row per measure (`fold: L` items from `axisItems`, computed like a subtotal but without the Total label; inner levels blank). Subtotals at that level only show when the group is open. Totals rules apply, so "Leave blank" measures are blank on closed rows.
   - The row-label header of each such level has Expand all / Collapse all (expand opens that level and those above it; collapse closes it and those below).
   - Open/closed state is session only: `folds = { depth, flip }` (levels below `depth` open; `flip` holds `"L:field=member|…"` exceptions). Start state: all closed.
   - Excel export and tooltips follow what's shown.

### 8.0.1 To explore later: on-demand fields (no query until used)

Goal: the field list shows many attributes without Tableau querying them; a field is only queried once a viewer drags it onto the grid.

- Why the Fields tile can't do this: every pill on any Marks card tile is part of the sheet's level of detail, so Tableau queries at the grain of all of them and sends that summary data to the extension. Long attribute lists multiply the row count toward the raw grain (30–40M rows). Measures only add columns, so they're cheap.
- List without querying: `worksheet.getDataSourcesAsync()` returns the data source's fields (name, role, type) without a query. The author would pick available fields in a Configure dialog (stored in `tableau.extensions.settings`), not as pills.
- Query on demand: the 1.17 library has **undocumented** marks-card editing functions on the worksheet:
  - `addMarksCardFieldsAsync(marksCardIndex, encodingType, fields, startIndex)`
  - `spliceMarksCardFieldsAsync(marksCardIndex, encodingType, startIndex, deleteCount, fields)`
  - `moveMarksCardFieldAsync(marksCardIndex, fromIndex, toIndex, fieldCount)`
  If they work, dragging a field onto the grid adds the pill to the Rows/Columns tile and Tableau queries exactly the displayed grain.
- Risks: likely authoring-only (may be refused for viewers on Cloud), the `fields` argument format is unknown, and undocumented APIs may change.
- First step: a probe panel under `?debug=1` that lists data source fields and tries those calls with each likely argument format, run once in Desktop/web edit and once as a Cloud viewer.
- Fallbacks: parameter-driven slots (current Grid Level approach), or accept the Fields tile cost for low-cardinality attributes.

### 8.1 Next: sorting

- Sort on every value column header and on every measure.
- Sort dimension members per field, A–Z or Z–A, from the pill or the header label.
- Sorting must be part of `layout`, so it survives refreshes and gets saved later.
- **Open design question to settle with the user:** how sorting by a value column works with nested row dimensions. The suggested default is to sort the innermost row level within each parent group and keep the outer groups in member order. Once subtotals exist, outer groups can sort by their subtotal.

### 8.2 Next: subtotals and grand totals

- The viewer turns them on with a toolbar button or a per-field toggle, for each axis and level.
- Totals are calculated in JS from the records. Each measure needs a total rule, configured by the author through the extension's Configure dialog and stored in `tableau.extensions.settings`:
  - `sum`, `min` and `max` total correctly from cell values (Bet Eur, Bet Qty and the Bonus measures are sums);
  - `ratio` of numerator / denominator, where the author puts both measures on Values and the extension calculates `SUM(num) / SUM(den)` at each total level;
  - `none`, for COUNTD, AVG and LOD-based measures, which leaves the total cell blank.
- The same machinery would let viewers remove a dimension from the grid.
- The manifest needs `<context-menu><configure-context-menu-item/></context-menu>`, and `initializeAsync({ configure })` needs a configure callback. See the old `index.html` for the dialog pattern (`?configure=1` and `displayDialogAsync`).

### 8.3 Next: Excel and CSV export

- Export exactly what's on screen, using the current layout and nested headers with merged cells.
- Bring over the old module's parameter header logic:
  - the `AXES` patterns (`Grid Level N`, `Metric Selector`) and their pairing through `slotOf` and `prefixOf`;
  - `buildHeaderMap` and `labelOf`, which take parameter display aliases;
  - `NONE_LABELS`;
  - `SORT_FIELD_PATTERN` for dropping sort-helper fields.
- Read parameters with `worksheet.getParametersAsync()`, or the viz-extension equivalent, and listen for `ParameterChanged`.
- Name the file `<dashboard or sheet name>_<YYYY-MM-DD>.xlsx`. Fall back to CSV if SheetJS fails to load.
- Write numbers to Excel as native values with the precision Tableau shows. The old `decimalsOf` already reads that precision from `formattedValue`.
- Downloads from inside the extension frame worked in the old dashboard extension, but confirm they also work for viz extensions.

### 8.4 Later: save layouts with Custom Views

- Plan: serialise `layout`, including sorts and totals, to JSON in a hidden string parameter such as `pLayoutState`, using `changeValueAsync`. Tableau Custom Views save parameter values, so saving a Custom View should save the layout. On load, read the parameter and apply it through `reconcile`.
- **Prove this early with a small test**, because it's the one piece that relies on Tableau behaving as expected. Check the string length limits and that changing the parameter doesn't trigger unwanted refresh loops.
- Fallbacks if it doesn't work: browser storage keyed by workbook and sheet, or a small backend keyed by user, with the user identified by a `USERNAME()` calculated field placed on the viz.

### 8.5 Later

- Row virtualization for large sheets.
- Mark selection back to Tableau with `selectMarksByValueAsync`, so dashboard actions keep working.
- Show the viewer's layout changes in the shelves when the data reloads.

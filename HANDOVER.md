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
pivot-grid/
  index.html                          The whole extension: HTML, CSS and JS in one file
  pivot-grid.trex                     Viz extension manifest
  tableau.extensions.1.latest.min.js  NOT in this folder yet: copy it in (version 1.17)
  HANDOVER.md                         This file
```

Reference files from the original export module, which live in the project root and not in `pivot-grid/`:

- `index.html` is the old Grid Export dashboard extension. Its parameter header logic will be reused for the export.
- `tableau-export.trex` is the old manifest.
- `tableau_extensions_1_latest_min.js` is the Extensions API library, version 1.17. It contains `getVisualSpecificationAsync`, `worksheetContent`, `getSummaryDataReaderAsync` and `SummaryDataChanged`. The page loads it as `./tableau.extensions.1.latest.min.js`, so rename the file when you copy it in.
- `xlsx_full_min.js` is SheetJS, for the export iteration. The old module loads it as `./xlsx.full.min.js`.

Hosting URL in the manifest: `https://guido-riebeek-yolo2.github.io/tableau_export/pivot-grid/index.html`

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
  dims:     [{ id, label, col, axis, members: [labels in sort order], index: Map(label -> memberIdx) }],
  measures: [{ id, label, col }],
  records:  [{ k: [memberIdx per dim, in model.dims order], m: [cell per measure] }],
  lookup:   Map("i,j,k" -> record),        // key = member indices joined in model.dims order
  dimPos:   Map(dimId -> position in model.dims),
  measById: Map(measId -> { ...measure, index }),
  dupes:    count of rows whose dim key repeated (first row kept)
}
```

- A cell is `{ f: formatted text, s: sort key }`. The sort key is `nativeValue` when it's a number, `getTime()` for a Date, and the formatted text otherwise.
- Members are sorted by sort key. Numbers sort numerically. Text uses `Intl.Collator` with `numeric: true`, which matches Tableau's natural sort ("1x2Network, 3Oaks, 7rings, 100HP Gaming").
- Field ids are the Tableau field names, such as `SUM(Bet Eur)` or `YEAR(Calendar Date)`. Display labels come from `stripAgg` for measures and `prettyDim` for dimensions (`YEAR(x)` becomes "Year of x").

### 4.2 Layout

```
layout = { rows: [ids], cols: [ids], measures: [measure ids, display order], hidden: [measure ids] }
```

- `MEAS = "__measures__"` is the Measure Names pseudo-field. It can sit at any position on either axis.
- `defaultLayout()` builds the author's layout from the encodings. It places `MEAS` according to `DEFAULT_MEASURES_AXIS` and `DEFAULT_MEASURES_AT`.
- `reconcile(old)` runs on every data refresh. It keeps the viewer's arrangement, adds new fields to their default axis, and drops fields that no longer exist.
- Every layout change goes through `commit(next, movedId)`. It skips no-op changes, re-renders, and briefly flashes the moved pill. The callers are `moveField`, `moveMeasure`, `toggleMeasure`, Swap and Reset.

### 4.3 Pivot (`axisTuples`, `firstDiff`, `runLens`)

- `axisTuples(levels, vis)` returns one tuple per header path. Each tuple holds member indices aligned with the levels. At the `MEAS` level it holds an index into `vis`, the visible measures. Only dimension combinations that exist in the data are produced, as in Tableau. Tuples are sorted lexicographically.
- `firstDiff(T)[i]` is the level at which tuple `i` first differs from tuple `i-1`.
- `runLens(fd, lvl)` gives the span length at each run start and 0 elsewhere. It drives `colspan` and `rowspan`.
- To read a cell value, build the lookup key by walking `model.dims` and reading each dimension's index from the row or column tuple using `where`. Then take `rec.m[vis[mi].index].f`.

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
- Any element with `data-drag` (the field id) and `data-kind` can be dragged. `dim` and `meas` items go to rows or cols. `measure` items can only be reordered within the Values shelf.
- A drag starts after 4px of movement. Without that movement it counts as a click, and clicking a measure pill toggles it hidden.
- `zones(kind)` is recomputed on every pointer move, so scrolling mid-drag works. The zones are:
  - the shelves, with pills laid out as `dir: "wrap"`;
  - a grid column zone covering the header area above `tr.lrow`, with `dir: "v"` because levels stack vertically;
  - a grid row zone covering the row-header area, with `dir: "h"`.
- `updateTarget` finds the insertion point as a `beforeId`, not a numeric index. That keeps it correct even when the grid hides the `MEAS` level. It also positions the orange `#marker`.
- Escape cancels a drag.
- Keyboard support on a focused pill: Left and Right reorder it, Up moves a field from Rows to Columns, Down moves it from Columns to Rows, and Space or Enter toggles a measure.

### 4.6 Tableau integration

- `init()` uses demo data if `?demo=1` is set, if the library is missing, or if `initializeAsync` takes longer than `INIT_TIMEOUT_S` (8 seconds). If `worksheetContent` is missing, the page shows "Add this as a viz extension", because it was loaded as a dashboard extension.
- `loadFromTableau()` works in this order:
  1. Calls `getVisualSpecificationAsync()`, takes `marksSpecifications[activeMarksSpecificationIndex].encodings`, and maps each `e.id` to an axis through `ENC_AXIS`.
  2. Reads the summary data with `getSummaryDataReaderAsync(undefined, { ignoreSelection: true, applyWorksheetFormatting: true })` and `getAllPagesAsync()`, always calling `releaseAsync()` afterwards.
  3. Matches each encoding field name to a column with `matchColumn`: an exact match first, then whitespace- and case-normalised, then with the aggregation stripped.
- `refreshFromTableau()` is tied to `SummaryDataChanged` and coalesces bursts of events into one load at a time.
- `?debug=1` shows a diagnostics panel with the encodings, the summary columns with their data types, unmatched fields, model stats and the layout JSON.

## 5. Unverified against real Tableau (check these first)

None of this has run inside Tableau yet. Only demo mode has been tested.

1. **Encoding field name vs. summary column name.** We assume `encodings[].field.name` matches `columns[].fieldName`, give or take aggregation. Check with `?debug=1`. If they differ, `encodings[].field` may have other properties to match on, such as an id or caption. Log the whole object.
2. **Encoding structure.** We assume several fields on one tile appear as several entries with the same `id`, in tile order. Verify, especially that the order matches the tile.
3. **Manifest.** Check that the `role-type` values `discrete-dimension` and `continuous-measure`, `<fields max-count>`, and `min-api-version 1.11` are accepted. Compare with Tableau's official viz extension samples, such as the Sankey sample in the `tableau/extensions-api` GitHub repo.
4. **Init detection in Desktop.** The 8-second timeout fallback could trigger by mistake on a slow Server, so it may need raising.
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
- Viewers can't remove a dimension from the grid. That needs re-aggregation (section 8.2).
- The layout lives only in memory and is lost on reload (section 8.4).
- There is no dark-mode styling, which is fine inside Tableau.

## 8. Roadmap

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

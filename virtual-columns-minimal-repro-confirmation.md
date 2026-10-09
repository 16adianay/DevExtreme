# Virtual Columns Minimal Repro Confirmation (TreeList, hidden band children)

## Scope
Follow-up confirmation for `dxTreeList`/`dxDataGrid` virtual column rendering issue with:
- `scrolling.columnRenderingMode: 'virtual'`
- band columns whose direct children are all runtime-hidden (`visible: false`)

No source-code fix is included in this update. This is investigation-only.

## Environment and build
- Repository: `16adianay/DevExtreme`
- Branch in this session: `copilot/fix-virtual-columns-rendering-issue`
- Build executed:
  - `pnpm install --frozen-lockfile`
  - `pnpm nx build:dev devextreme`
- Runtime repro method: headless Chromium via Puppeteer (local one-off script in `/tmp`, not committed)

## Repro configuration used
Exact minimal config from the issue statement was used:
- 1 fixed real column (`col1`, width `150`)
- For `i = 2..40 step 2`:
  - real column `col{i}` width `125`
  - band column `Band{i}` with one child `visible: false`
- Tail real columns: `col50..col54` (all width `125`)
- Auto-scroll far right via `rowsView.getScrollable().scrollTo({ left: 100000 })`

## Captured runtime values at far-right scroll (baseline)

### Virtual range indices
- `_beginPageIndex = 7`
- `_endPageIndex = 9`
- `columnPageSize = 5`
- `startIndex = 35`
- `endIndex = 45`

So the rendered leaf range is `[35, 45)`.

### Visible leaf columns / pseudo-leaf bands
- `getVisibleColumns(undefined, true)` count: `46`
- Band columns present in that visible leaf list as leaf-like entries (`isBand: true`, `colspan: 0`): `20`
  - `Band2, Band4, ..., Band40`

This confirms the hidden-children band columns are participating in the leaf-width pipeline with `colspan === 0`.

### Widths (assumed vs DOM)
- Assumed total width (same fallback logic as `getColumnWidths`: `visibleWidth || width`, else `50`): `4275`
- DOM rows-view scroll content width (`.dx-treelist-rowsview .dx-scrollable-content.scrollWidth`): `4125`
- Divergence: `+150` px (assumed minus actual)

Additional decomposition:
- The 20 pseudo-leaf `Band{i}` columns contribute exactly `20 * 50 = 1000` px to the assumed width path.
- Compared with mitigation run (below), baseline DOM `scrollWidth` is larger by `850` px, while assumed width is larger by full `1000` px, producing the observed `150` px drift.

### col53/col54 inclusion and DOM evidence
- `col53` visible index: `44` -> **inside** `[35, 45)`
- `col54` visible index: `45` -> **outside** `[35, 45)`
- DOM text present:
  - `"Value 1-53"`: **present**
  - `"Value 1-54"`: **absent**

Observed rendered first data row tail confirms last visible data stops at `col53`, with reserved but empty trailing slot for `col54`.

## Mitigation confirmation: band `width: 0`
Second run used the same config, only adding `width: 0` on each `Band{i}`.

Captured results:
- `_beginPageIndex = 7`, `_endPageIndex = 10`
- `startIndex = 35`, `endIndex = 50`
- `col54` index `45` is now **inside** `[35, 50)`
- `"Value 1-54"` is **present** in DOM
- Assumed width: `3275`
- DOM `scrollWidth`: `3275`
- Divergence: `0`

Conclusion: `width: 0` on these zero-visible-children band columns resolves this minimal repro as well.

## Final root-cause statement (for official bug report)
When a band column ends up with zero visible direct children (e.g., all hidden through Column Chooser), `calculateColspan` returns `0` because it uses visible-only child lookup (`getChildrenByBandColumn(columnID, true)`) in:
- `packages/devextreme/js/__internal/grids/grid_core/columns_controller/m_columns_controller_utils.ts`
  - `calculateColspan`

Then `processBandColumns` treats that band as leaf-like due to falsy `colspan` (`!column.colspan`):
- same file, `processBandColumns`

Those leaf-like band entries are included in virtual width/range computation where missing numeric width falls back to `DEFAULT_COLUMN_WIDTH = 50`:
- `packages/devextreme/js/__internal/grids/grid_core/m_utils.ts`
  - `getColumnWidths`

The resulting width model feeds page-index boundaries used by virtual column compilation:
- `packages/devextreme/js/__internal/grids/grid_core/virtual_columns/m_virtual_columns.ts`
  - `getBeginPageIndex`
  - `getEndPageIndex`
  - `_compileVisibleColumns` (`startIndex = beginPageIndex * columnPageSize`, `endIndex = endPageIndex * columnPageSize`)

In this minimal repro, the drift is enough that the final real leaf column (`col54`) falls outside `[startIndex, endIndex)` at far-right scroll, while `col53` remains included.

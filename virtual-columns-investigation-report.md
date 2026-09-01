# Virtual Column Rendering Investigation (`26_1`)

## 1) Repro status

**Confirmed.** The customer configuration reproduces on this branch for **both** `dxTreeList` (exact requested repro) and `dxDataGrid`.

At far-right scroll (`scrollLeft: 3602`), the last data column (`col129`) is outside `_compileVisibleColumns` range:

- `beginPageIndex = 6`
- `endPageIndex = 8`
- `columnPageSize = 5`
- `startIndex = 30`
- `endIndex = 40`
- last visible leaf data column index = `40`
- condition: `40 >= 30 && 40 < 40` -> **false**

So space is reserved, but the final leaf column content is not rendered.

## 2) Root-cause mechanism (with runtime evidence)

The issue is caused by **width-model divergence** used by virtual column paging:

- Virtual range calculation uses `getColumnWidths` (`m_utils.ts`) with fallback `DEFAULT_COLUMN_WIDTH = 50` for columns without numeric width.
- In this config, many `columns: []` objects are treated as visible leaf columns (no `dataField`) and contribute fallback width in calculation.
- Actual rendered scrollable width at runtime is smaller than the assumed width used for range computation.

Measured in customer repro:

- `totalAssumedWidth` (from virtual calc model): **4550**
- actual scrollable content width (DOM): **4450**
- mismatch causes right-edge page calculation to stop early (`endIndex=40`), excluding final leaf (`index=40`).

Pseudo-leaf columns detected in the customer config (non-`dataField`, `columns: []`): **10** entries, most using fallback width assumption 50.

This is why repro is intermittent: page boundaries are quantized by `columnPageSize` (5), so only certain visibility/count combinations push the last real leaf exactly beyond `[startIndex, endIndex)`.

## 3) Minimal trigger configuration

Using a reduced synthetic banded config (same virtual settings):

- fixed left band with 3 real columns,
- metrics band with `leafCount = 8` real leaves,
- plus `pseudoCount = 5` empty pseudo-columns (`columns: []`, no `dataField`, no explicit width),

produces:

- `startIndex = 5`, `endIndex = 15`, `lastLeafIndex = 15`
- last leaf excluded (`15 < 15` false)

This is the smallest far-right case in the focused sweep that reliably demonstrates the same mechanism with non-zero horizontal scroll.

## 4) Causality checks (requested)

### A. Set explicit widths on **band columns only**

Result: **does not fix**.

Customer repro with band widths added:

- still `startIndex = 30`, `endIndex = 40`, `lastLeafIndex = 40` (excluded)

### B. Remove empty `columns: []` pseudo-columns

Result: **fixes** the issue.

Customer repro after removing pseudo-columns:

- `startIndex = 25`, `endIndex = 35`, `lastLeafIndex = 30` (included)
- last-column data renders.

### C. Keep pseudo-columns but set explicit width `0`

Result: **fixes** the issue.

Customer repro with width `0` on empty pseudo-columns:

- `startIndex = 30`, `endIndex = 45`, `lastLeafIndex = 40` (included)
- assumed width and actual scrollable width align (`3900` vs `3900`).

## 5) Final conclusion

Root cause is **not** primarily “band columns missing width”.

Primary trigger is the presence of empty `columns: []` pseudo-columns (no `dataField`) participating in virtual width accumulation with fallback/default width assumptions that diverge from actual rendered layout width. The quantized page-window math (`columnPageSize`) makes the symptom threshold-based/intermittent and most visible at far-right scroll.

## 6) Investigation artifacts

Investigation was performed with runtime instrumentation in a temporary local repro harness (headless browser), collecting:

- `_beginPageIndex`, `_endPageIndex`, `startIndex`, `endIndex`
- last leaf index inclusion check against `[startIndex, endIndex)`
- assumed width model total vs DOM scrollable content width
- per-scenario mitigation outcomes

No production source fix was implemented in this branch (investigation-only as requested).

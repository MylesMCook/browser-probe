# Data Display Patterns

---

Before using this file:

- Take a fresh `agent-browser snapshot -i -C`.
- Run the preflight in [../session-preflight.md](../session-preflight.md) before any `eval`.
- Prefer `get count`, `get text`, and `get url` when they answer the question directly.

## Detect data display type

Before running any data display tests, establish what kind of display is in use:

```bash
# Run the preflight in ../session-preflight.md first.
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  hasTables: document.querySelectorAll('table tbody tr').length,
  hasCards: document.querySelectorAll('[class*="card"], [class*="Card"]').length,
  hasListItems: document.querySelectorAll('[class*="list-item"], [class*="item"], li[class]').length,
  isVirtualized: !!(
    document.querySelector('[data-virtuoso-scroller], [class*="virtualized"], [class*="virtual"], [style*="transform: translateY"]') ||
    document.querySelector('[data-rowindex], [aria-rowindex]')
  ),
  hasPagination: !!document.body.textContent.match(/\d+\s*[-–]\s*\d+\s+of\s+[\d,]+/),
  // Only flag as explicit infinite scroll if there is a deliberate signal — not just absence of pagination text
  hasExplicitInfiniteScroll: !!(
    document.querySelector('[data-infinite-scroll], [class*="infinite-scroll"], [class*="InfiniteScroll"]') ||
    document.querySelector('[class*="load-more"], [class*="loadMore"]')
  )
})
EVALEOF
```

Use the result to decide which sections to run:

- `hasTables > 0` and not `isVirtualized` and `hasPagination` → run all table sections normally
- `hasTables > 0` and not `isVirtualized` and not `hasPagination` and `hasExplicitInfiniteScroll` → run **Infinite scroll** section instead of pagination
- `hasTables > 0` and not `isVirtualized` and not `hasPagination` and not `hasExplicitInfiniteScroll` → the table probably uses cursor-based or button pagination without an "N of M" string — still run the pagination section but use Prev/Next buttons rather than the range text assertion
- `isVirtualized: true` → run the **Virtualized list** section below instead of normal row count assertions
- Only `hasCards` or `hasListItems` → skip table-specific sections; run card/metric sections

---

## Virtualized list — data exists and renders correctly

When `isVirtualized` is true, `querySelectorAll` returns only the currently visible rows (typically 10-30), not the full dataset. Never assert row count as a proxy for "data loaded."

Scroll down to trigger more renders. Virtualized lists use their own scroll container — scrolling the window does nothing:

```bash
agent-browser eval --stdin <<'EVALEOF'
// Find the actual scroll container — could be a div, not the window
const scrollContainer =
  document.querySelector('[data-virtuoso-scroller]') ||
  document.querySelector('[class*="virtualized"], [class*="virtual"]') ||
  document.querySelector('[aria-rowcount]')?.closest('[style*="overflow"]') ||
  // Fall back: find the tallest scrollable element
  Array.from(document.querySelectorAll('*')).filter(el => {
    const s = window.getComputedStyle(el);
    return (s.overflow === 'auto' || s.overflow === 'scroll' || s.overflowY === 'auto' || s.overflowY === 'scroll') &&
      el.scrollHeight > el.clientHeight + 50;
  }).sort((a, b) => b.scrollHeight - a.scrollHeight)[0];

if (scrollContainer) {
  scrollContainer.scrollTop = scrollContainer.scrollHeight / 2;
  JSON.stringify({ scrolled: true, containerTag: scrollContainer.tagName, scrollHeight: scrollContainer.scrollHeight });
} else {
  window.scrollTo(0, document.body.scrollHeight / 2);
  JSON.stringify({ scrolled: true, containerTag: 'window', scrollHeight: document.body.scrollHeight });
}
EVALEOF
```

```bash
agent-browser wait 500
agent-browser eval --stdin <<'EVALEOF'
const scrollContainer =
  document.querySelector('[data-virtuoso-scroller]') ||
  document.querySelector('[class*="virtualized"], [class*="virtual"]') ||
  Array.from(document.querySelectorAll('*')).filter(el => {
    const s = window.getComputedStyle(el);
    return (s.overflowY === 'auto' || s.overflowY === 'scroll') && el.scrollHeight > el.clientHeight + 50;
  }).sort((a, b) => b.scrollHeight - a.scrollHeight)[0];
if (scrollContainer) scrollContainer.scrollTop = scrollContainer.scrollHeight;
else window.scrollTo(0, document.body.scrollHeight);
true
EVALEOF
```

```bash
agent-browser wait 500
agent-browser screenshot --full
```

Assert data is actually loading as you scroll:

```bash
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  visibleRowCount: document.querySelectorAll('table tbody tr, [data-rowindex], [aria-rowindex]').length,
  totalIndicator: document.body.textContent.match(/[\d,]+\s+(rows?|records?|items?|results?)/i)?.[0],
  lastRowContent: Array.from(document.querySelectorAll('table tbody tr, [aria-rowindex]')).pop()?.textContent?.trim()?.substring(0, 60)
})
EVALEOF
```

Expected: `visibleRowCount` > 0, `lastRowContent` is real data (not empty or loading spinner). If `totalIndicator` exists, note the total for reference. **PASS / FAIL**

Scroll back to top and verify first row is still correct:

```bash
agent-browser eval --stdin <<'EVALEOF'
const scrollContainer =
  document.querySelector('[data-virtuoso-scroller]') ||
  Array.from(document.querySelectorAll('*')).filter(el => {
    const s = window.getComputedStyle(el);
    return (s.overflowY === 'auto' || s.overflowY === 'scroll') && el.scrollHeight > el.clientHeight + 50;
  }).sort((a, b) => b.scrollHeight - a.scrollHeight)[0];
if (scrollContainer) scrollContainer.scrollTop = 0;
else window.scrollTo(0, 0);
true
EVALEOF
```

```bash
agent-browser wait 500
agent-browser get text 'table tbody tr td, [aria-rowindex]'
```

Expected: renders correctly at top — not blank, not showing last-page data. **PASS / FAIL**

---

## Infinite scroll — loads more on scroll

When `hasExplicitInfiniteScroll` is true and there is no traditional pagination:

Note initial item count:

```bash
agent-browser get count 'table tbody tr, [class*="card"], [class*="item"]'
```

Scroll to bottom and wait for new content. Use the scroll container, not the window:

```bash
agent-browser eval --stdin <<'EVALEOF'
const container =
  document.querySelector('[data-infinite-scroll], [class*="infinite-scroll"]') ||
  Array.from(document.querySelectorAll('*')).filter(el => {
    const s = window.getComputedStyle(el);
    return (s.overflowY === 'auto' || s.overflowY === 'scroll') && el.scrollHeight > el.clientHeight + 50;
  }).sort((a, b) => b.scrollHeight - a.scrollHeight)[0];
if (container) container.scrollTop = container.scrollHeight;
else window.scrollTo(0, document.body.scrollHeight);
true
EVALEOF
```

```bash
agent-browser wait 1500
agent-browser screenshot --full
agent-browser get count 'table tbody tr, [class*="card"], [class*="item"]'
```

Expected: count is greater than before — new items loaded. If count stayed the same and there's no "end of results" indicator, infinite scroll is broken. That's a High finding. **PASS / FAIL**

Check for end-of-list signal after scrolling to true bottom:

```bash
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  hasEndSignal: !!document.querySelector('[class*="end"], [class*="no-more"], [class*="all-loaded"]') ||
    document.body.textContent.includes('No more') || document.body.textContent.includes('End of'),
  hasLoadingSpinner: !!document.querySelector('[class*="spinner"], [class*="loading"], [role="progressbar"]')
})
EVALEOF
```

Expected: either an end signal or a loading spinner while fetching — not silence. **PASS / FAIL**

---

## Verify table structure and data loaded

```bash
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  headers: Array.from(document.querySelectorAll('table th, [role="columnheader"]')).map(h => h.textContent?.trim()),
  rowCount: document.querySelectorAll('table tbody tr, [role="row"]:not([role="columnheader"])').length,
  hasData: document.querySelectorAll('table tbody tr').length > 0
})
EVALEOF
```

Expected: headers non-empty, `rowCount` > 0. **PASS / FAIL**

Spot-check first 3 rows for non-empty cells:

```bash
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify(Array.from(document.querySelectorAll('table tbody tr')).slice(0, 3).map(row =>
  Array.from(row.querySelectorAll('td')).map(cell => cell.textContent?.trim()).filter(Boolean)
))
EVALEOF
```

Expected: each row has meaningful content — not all-empty, not all "--" or "null". **PASS / FAIL**

---

## Sortable columns — clicking reorders data

```bash
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify(Array.from(document.querySelectorAll('th button, [aria-sort], th[class*="sort"]')).map(h => h.textContent?.trim()))
EVALEOF
```

If sortable headers found — check row count first, then click a header twice (ascending then descending) to guarantee an observable change regardless of initial sort order:

```bash
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  rowCount: document.querySelectorAll('table tbody tr').length,
  firstCellValue: document.querySelector('table tbody tr td')?.textContent?.trim(),
  lastCellValue: Array.from(document.querySelectorAll('table tbody tr td:first-child')).pop()?.textContent?.trim()
})
EVALEOF
```

If `rowCount` is 0 or 1: skip this test — there is not enough data to confirm sort works. Note in report as "not enough data to test sort."

If `rowCount` >= 2: click the sortable header once, wait, then click it again (forces direction change):

```bash
agent-browser snapshot -i -C
# Click sortable header first time
agent-browser click @eN
agent-browser wait 1000
agent-browser get text 'table tbody tr td'
# Note value A

agent-browser snapshot -i -C
# Click the same header again to reverse sort
agent-browser click @eN
agent-browser wait 1000
agent-browser get text 'table tbody tr td'
# Note value B
```

Expected: value A ≠ value B — the first row changed between the two sort directions. If both values are identical the sort is not working. **PASS / FAIL**

---

## Pagination — next and previous work correctly

```bash
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  pageText: document.body.textContent.match(/(\d+)\s*[-–]\s*(\d+)\s+of\s+([\d,]+)/)?.[0],
  // Detect next/prev by aria-label, icon class, or disabled state — not English text
  // aria-label is language-agnostic; SVG icon class names (ChevronRight etc.) are too
  hasNext: !!Array.from(document.querySelectorAll('button, a')).find(el => {
    const label = (el.getAttribute('aria-label') || '').toLowerCase();
    const cls = el.className || '';
    const text = (el.textContent || '').trim().toLowerCase();
    return /next|›|forward|chevron.?right|arrow.?right|siguiente|suivant|weiter|avanti/i.test(text + ' ' + label + ' ' + cls)
      && !el.disabled;
  }),
  hasPrev: !!Array.from(document.querySelectorAll('button, a')).find(el => {
    const label = (el.getAttribute('aria-label') || '').toLowerCase();
    const cls = el.className || '';
    const text = (el.textContent || '').trim().toLowerCase();
    return /prev|‹|back|chevron.?left|arrow.?left|anterior|précédent|zurück|precedente/i.test(text + ' ' + label + ' ' + cls)
      && !el.disabled;
  })
})
EVALEOF
```

Click Next — verify range advances:

```bash
agent-browser snapshot -i -C
# Identify Next button ref
agent-browser click @eN
agent-browser wait --load networkidle
agent-browser eval 'document.body.textContent.match(/(\d+)\s*[-–]\s*(\d+)\s+of/)?.[0]'
```

Expected: range start is greater than before (e.g. "1-25" → "26-50"). **PASS / FAIL**

Hypothesis check — does sort reset pagination to page 1? Run this only if the Next step above succeeded (you're now on page 2). If the app has sortable columns:

```bash
agent-browser snapshot -i -C
# Click any sortable column header while on page 2
agent-browser click @eN
agent-browser wait 1000
agent-browser eval 'document.body.textContent.match(/(\d+)\s*[-–]\s*(\d+)\s+of/)?.[1]'
```

Expected: `"1"` — sorting from any page should return to page 1. If it returns `"2"` or higher, you're seeing rows 26–50 sorted differently with no indication — that's a Medium finding. If the Next test was skipped or failed, skip this test. **PASS / FAIL / SKIPPED**

Click Prev — must be on page 2 for this to be testable. If the sort-reset check above moved you back to page 1, navigate to page 2 again before running this:

```bash
agent-browser snapshot -i -C
# If on page 1 after sort-reset: click Next to get back to page 2, then continue
agent-browser click @eN
agent-browser wait --load networkidle
```

Now click Prev:

```bash
agent-browser snapshot -i -C
# Identify Prev button ref
agent-browser click @eN
agent-browser wait --load networkidle
agent-browser eval 'document.body.textContent.match(/(\d+)\s*[-–]\s*(\d+)\s+of/)?.[1]'
```

Expected: `"1"` — back to the first page. If Prev navigates to the wrong page or the Prev button was disabled on page 2, that's a High finding. If you never successfully navigated to page 2 in the Next step above, skip this test. **PASS / FAIL / SKIPPED**

---

## Row click navigation

```bash
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  // Check multiple signals — Tailwind sets cursor-pointer on cells, some apps use onClick with no class
  hasAnchorRows: !!document.querySelector('table tbody tr a, table tbody td a'),
  hasButtonRows: !!document.querySelector('table tbody tr button'),
  hasCursorRows: !!document.querySelector('table tbody tr[class*="cursor"], table tbody td[class*="cursor-pointer"]'),
  // Snapshot will reveal onClick divs — this is a hint to look at the snapshot carefully
  rowCount: document.querySelectorAll('table tbody tr').length
})
EVALEOF
```

If any signal is true, OR if the snapshot shows interactive row refs: rows are likely clickable. Also check the snapshot directly — rows with `onClick` handlers show as refs in `snapshot -i -C` even without anchor or cursor classes.

If rows are clickable:

```bash
agent-browser get text 'table tbody tr td'
# note this value — verify the detail page matches

agent-browser snapshot -i -C
# Identify a row or cell link ref
agent-browser click @eN
agent-browser wait --load networkidle
agent-browser get url
agent-browser screenshot --full
```

Expected: navigated to a detail page relevant to the clicked row — not a different record or a 404. **PASS / FAIL**

---

## Empty state is friendly

To reach an empty state, try one of these approaches in order:

1. If the app has a section that might have no data by default (an "Archived" tab, a filtered view, a fresh account with no records) — navigate there.
2. If the search-filter pattern has already been run: the no-results search already produced a zero-result state — assess the empty state shown there.
3. If neither applies: apply a filter so extreme it returns nothing (e.g. apply a date range far in the future), then check the state below.

```bash
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  hasEmptyState: !!(
    document.querySelector('[class*="empty" i], [class*="no-data" i], [class*="zero-state" i]') ||
    document.querySelector('[class*="EmptyState"], [class*="NoResults"]') ||
    document.querySelector('[class*="blank-slate"], [class*="placeholder"]')
  ),
  emptyText: (
    document.querySelector('[class*="empty" i], [class*="no-data" i], [class*="EmptyState"]')
  )?.textContent?.trim()?.substring(0, 100),
  rowCount: document.querySelectorAll('table tbody tr, [class*="card"], [class*="item"]').length,
  hasRawError: document.body.textContent.includes('undefined') ||
    document.body.textContent.includes('null') ||
    document.body.textContent.includes('NaN')
})
EVALEOF
```

Expected: `rowCount` is 0, `hasEmptyState` true with friendly text (not a spinner), `hasRawError` false. Raw "undefined" or "null" visible to users is a Medium finding. If `hasEmptyState` is false but `rowCount` is 0: no empty state UI exists — that's a Low finding (blank area with no guidance). **PASS / FAIL**

---

## Metric cards — values are populated, not placeholder

```bash
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify(Array.from(document.querySelectorAll('[class*="metric"], [class*="stat"], [class*="kpi"], [class*="card"]')).slice(0, 8).map(card => ({
  label: card.querySelector('[class*="label"], [class*="caption"], p')?.textContent?.trim(),
  value: card.querySelector('[class*="value"], h2, h3, strong, [class*="heading"]')?.textContent?.trim()
})).filter(c => c.label || c.value))
EVALEOF
```

Expected: values are numbers or formatted strings — not "...", "--", "NaN", "undefined", or "0" across every card simultaneously. All zeros could be real data, or it could indicate a failed API call — use judgment. **PASS / FAIL**

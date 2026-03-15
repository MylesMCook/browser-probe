# Search & Filter Patterns

---

## Discover search and filter controls

```bash
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  searchInputs: Array.from(document.querySelectorAll([
    'input[type="search"]',
    'input[placeholder*="search" i]',
    'input[placeholder*="filter" i]',
    'input[placeholder*="find" i]',
    'input[aria-label*="search" i]',
    'input[role="searchbox"]',
    '[role="search"] input',
    'input[placeholder*="suche" i]',
    'input[placeholder*="recherche" i]',
    'input[placeholder*="buscar" i]',
    'input[placeholder*="cerca" i]'
  ].join(', '))).map(i => ({
    placeholder: i.placeholder,
    ariaLabel: i.getAttribute('aria-label'),
    type: i.type
  })),
  filterSelects: Array.from(document.querySelectorAll('select')).map(s => ({ id: s.id, optionCount: s.options.length })),
  filterButtons: Array.from(document.querySelectorAll('button, [role="button"]')).filter(b =>
    /filter|sort|view|filtr|trier|filtrar/i.test(b.textContent + b.getAttribute('aria-label'))
  ).map(b => b.textContent?.trim())
})
EVALEOF
```

---

## Inline search — returns matching results

First, find a real term from the current data to use as the search query:

```bash
agent-browser eval --stdin <<'EVALEOF'
const cells = Array.from(document.querySelectorAll('table tbody tr td:first-child, [class*="card"] [class*="title"], [class*="card"] [class*="name"], [class*="item"] span'));
const terms = cells.map(el => el.textContent?.trim()).filter(t => t && t.length > 2 && t.length < 30);
JSON.stringify({ firstTerm: terms[0], termCount: terms.length })
EVALEOF
```

Use the returned `firstTerm` as the search query below. If `termCount` is 0, the search cannot be meaningfully tested yet — note this and skip to the next section.

Note the unfiltered count first:

```bash
agent-browser eval 'document.querySelectorAll("table tbody tr, [class*=\"card\"], [class*=\"item\"]").length'
```

Re-snapshot, fill search with the real term found above — substitute the actual `firstTerm` value returned by the discovery eval:

```bash
agent-browser snapshot -i -C
# Identify search input ref, e.g. @e3 [input placeholder="Search..."]
# IMPORTANT: replace the fill value with the actual firstTerm returned above
agent-browser fill @eN "SUBSTITUTE_firstTerm_HERE"
agent-browser wait 600
agent-browser screenshot --full
```

Assert filtered count is less and results are relevant:

```bash
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  count: document.querySelectorAll('table tbody tr, [class*="card"], [class*="item"]').length,
  firstFewResults: Array.from(document.querySelectorAll('table tbody tr td:first-child, [class*="card"] [class*="title"]')).slice(0, 5).map(el => el.textContent?.trim())
})
EVALEOF
```

Expected: `count` < unfiltered count, visible results relate to the search term. **PASS / FAIL**

---

## Search with no results — empty state, not broken UI

```bash
agent-browser snapshot -i -C
agent-browser fill @eN "zzznonexistentvalue99887"
agent-browser wait 600
agent-browser screenshot --full
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  rowCount: document.querySelectorAll('table tbody tr').length,
  hasEmptyState: !!document.querySelector('[class*="empty" i], [class*="no-results" i], [class*="zero" i]'),
  hasRawError: document.body.textContent.includes('undefined') || document.body.textContent.includes('Cannot read')
})
EVALEOF
```

Expected: `rowCount` 0, `hasEmptyState` true, `hasRawError` false. **PASS / FAIL**

---

## Clear search — full list restores

React controlled inputs often don't respond to `fill @ref ""` — they need the field selected and then cleared:

```bash
agent-browser snapshot -i -C
# Identify the search input ref
agent-browser click @eN
agent-browser wait 100
# Select all and delete — try Control+a (Windows/Linux) first
agent-browser press "Control+a"
agent-browser press "Backspace"
agent-browser wait 500
agent-browser eval 'document.querySelectorAll("table tbody tr, [class*=\"card\"], [class*=\"item\"]").length'
```

If count is still filtered, try the macOS shortcut:

```bash
agent-browser press "Meta+a"
agent-browser press "Backspace"
agent-browser wait 500
agent-browser eval 'document.querySelectorAll("table tbody tr, [class*=\"card\"], [class*=\"item\"]").length'
```

If still filtered, try Escape (some search implementations use it to clear):

```bash
agent-browser press Escape
agent-browser wait 500
agent-browser eval 'document.querySelectorAll("table tbody tr, [class*=\"card\"], [class*=\"item\"]").length'
```

Expected: count returns to the original unfiltered count. **PASS / FAIL**

---

## Special character handling

```bash
agent-browser snapshot -i -C
agent-browser fill @eN "test & search % query"
agent-browser wait 600
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  pageErrored: document.body.textContent.includes('Error') && document.body.textContent.includes('500'),
  urlEncoded: window.location.href,
  hasResults: document.querySelectorAll('table tbody tr, [class*="card"]').length
})
EVALEOF
```

Expected: no server error (500), URL is properly encoded (no raw `&` breaking query params), app stays functional. If special characters cause a server error or JS exception, that's a High finding. **PASS / FAIL**

---

## Multi-filter — applying filters narrows results progressively

First discover available filter controls and their options:

```bash
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  nativeSelects: Array.from(document.querySelectorAll('select')).map(s => ({
    id: s.id || s.name,
    options: Array.from(s.options).map(o => o.text).filter(t => t && !/all|any|select/i.test(t)).slice(0, 3)
  })).filter(s => s.options.length > 0),
  hasPortalledFilters: !!(
    document.querySelector('[role="combobox"][aria-haspopup="listbox"]') ||
    document.querySelector('[class*="MuiSelect"], [class*="ant-select"]')
  )
})
EVALEOF
```

**If `hasPortalledFilters` is true:** do not use `agent-browser select` — click the combobox trigger, wait, re-snapshot, then click the option ref. Repeat for each filter.

Note count after applying filter 1 using a real option value from discovery. SUBSTITUTE the actual option text from `nativeSelects` above before executing:

```bash
agent-browser snapshot -i -C
# Apply first filter — SUBSTITUTE_first_option_HERE with a real value from discovery
# e.g. if discovery returned options ["Active", "Inactive"], use "Active"
agent-browser select @eN "SUBSTITUTE_first_option_HERE"
agent-browser wait 500
agent-browser eval 'document.querySelectorAll("table tbody tr, [class*=\"card\"]").length'
# record count-after-filter-1
```

Apply filter 2 — count should narrow further. SUBSTITUTE the second option:

```bash
agent-browser snapshot -i -C
# SUBSTITUTE_second_option_HERE with a real value from a different filter's discovery
agent-browser select @eN "SUBSTITUTE_second_option_HERE"
agent-browser wait 500
agent-browser eval 'document.querySelectorAll("table tbody tr, [class*=\"card\"]").length'
```

Expected: count ≤ count-after-filter-1. If applying a second filter *increases* the count, the filters are likely using OR logic where AND is expected — that's a High finding. **PASS / FAIL**

---

## Clear all filters — full list restores

```bash
agent-browser eval --stdin <<'EVALEOF'
// Look for clear/reset button by text (English and common translations) or by common class patterns
const clearBtn = Array.from(document.querySelectorAll('button, [role="button"]')).find(b => {
  const text = (b.textContent || '').trim();
  const label = b.getAttribute('aria-label') || '';
  const combined = (text + ' ' + label).toLowerCase();
  const cls = b.className || '';
  return /clear all|clear filters|reset|remove filters|zurücksetzen|réinitialiser|limpiar/i.test(combined) ||
    /clear-filter|resetFilter|clearAll/i.test(cls);
});
clearBtn?.textContent?.trim() || null
EVALEOF
```

If a clear button exists:

```bash
agent-browser snapshot -i -C
agent-browser click @eN
agent-browser wait 500
agent-browser eval 'document.querySelectorAll("table tbody tr, [class*=\"card\"]").length'
```

Expected: count returns to unfiltered total. **PASS / FAIL**

---

## Filter state in URL — filtered views are shareable

After applying a filter, check the URL:

```bash
agent-browser get url
```

If the URL contains filter parameters: copy the URL returned above, then navigate fresh to it:

```bash
# Paste the URL captured above, replacing PASTE_FILTERED_URL_HERE:
agent-browser open "PASTE_FILTERED_URL_HERE"
agent-browser wait --load networkidle
agent-browser eval 'document.querySelectorAll("table tbody tr, [class*=\"card\"]").length'
```

Expected: same filtered count as before. If the URL has no filter params and navigating to the page loses filter state, that's a Medium finding — filtered views can't be shared or bookmarked. **PASS / FAIL**

---

## Global search — finds content and navigates to it

If a site-wide search exists (triggered by icon click or keyboard shortcut):

```bash
# Try Ctrl+K (Windows/Linux) first, then Cmd+K (macOS) if that doesn't open anything
agent-browser press "Control+k"
agent-browser wait 400
agent-browser screenshot --full
agent-browser eval '!!(document.querySelector("[role=\"dialog\"], [role=\"combobox\"][aria-expanded=\"true\"], [class*=\"command\"], [class*=\"spotlight\"]"))'
```

If the result was `false`, try the macOS shortcut:

```bash
agent-browser press "Meta+k"
agent-browser wait 400
agent-browser screenshot --full
```

If neither shortcut worked, find and click the search trigger:

```bash
agent-browser snapshot -i -C
# Identify search icon / "Search..." bar ref in the snapshot
agent-browser click @eN
agent-browser wait 400
```

Type a real term from the data. Use the same `firstTerm` discovered earlier in the inline search section — substitute its actual value, not the placeholder text:

```bash
agent-browser snapshot -i -C
# Identify search input inside the search panel
# IMPORTANT: replace SUBSTITUTE_firstTerm_HERE with the actual term from earlier discovery
agent-browser fill @eN "SUBSTITUTE_firstTerm_HERE"
agent-browser wait 800
agent-browser screenshot --full
agent-browser snapshot -i -C
# Identify first result ref
agent-browser click @eN
agent-browser wait --load networkidle
agent-browser get url
agent-browser screenshot --full
```

Expected: navigated to a relevant page — not homepage or a 404. **PASS / FAIL**

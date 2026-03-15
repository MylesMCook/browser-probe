# Navigation Patterns

---

Before using this file:

- Take a fresh `agent-browser snapshot -i -C`.
- Run the preflight in [../session-preflight.md](../session-preflight.md) before any `eval`.
- Prefer `get url`, `get text`, and `wait --url` when they answer the question directly.

## Discover all nav links and their destinations

```bash
# Run the preflight in ../session-preflight.md first.
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify(Array.from(document.querySelectorAll('nav a, [role="navigation"] a, header a')).map(a => ({
  text: a.textContent?.trim(),
  href: a.getAttribute('href')
})).filter(a => a.text && a.href))
EVALEOF
```

---

## For each nav link: click, wait, assert URL and heading

Run this for each link in the nav. After each test, navigate back to the starting URL before testing the next link — the nav structure may differ on the destination page.

```bash
agent-browser snapshot -i -C
# Identify link ref from snapshot — e.g. @e3 [link] "Settings"
agent-browser click @eN
agent-browser wait --load networkidle
agent-browser get url
agent-browser get text 'h1, h2'
agent-browser screenshot --full
# Navigate back to starting point before next link test:
agent-browser open "$BASE_URL"
agent-browser wait --load networkidle
```

Expected: URL matches the link's href. Heading reflects the destination. **PASS / FAIL**

---

## Active state updates on click

Navigate to a non-home page first, then click a different nav link — this ensures the test can actually observe the active state change rather than confirming the home link is trivially active on the home page. Use any non-home route from the app model. Substitute the actual path before executing (e.g. if the app model lists `/settings`, set it there):

```bash
# SUBSTITUTE the path: replace ROUTE_PATH below with a real route from the app model
# e.g. agent-browser open "$BASE_URL/settings"
agent-browser open "$BASE_URL/ROUTE_PATH"
agent-browser wait --load networkidle
agent-browser snapshot -i -C
# Click a nav link that is NOT the current page
agent-browser click @eN
agent-browser wait --load networkidle
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  ariaCurrent: document.querySelector('[aria-current="page"]')?.textContent?.trim(),
  activeByClass: Array.from(document.querySelectorAll('nav a, [role="navigation"] a')).find(a =>
    /active|selected|current/.test(a.className)
  )?.textContent?.trim()
})
EVALEOF
```

Expected: exactly one link is marked active and it matches the page you navigated to — not the page you came from. **PASS / FAIL**

---

## Active state on direct URL navigation (not just clicks)

Navigate directly to a route rather than clicking to it. Use any route from the app model. Substitute the actual path before executing (e.g. `/settings`, `/users`, `/dashboard`):

```bash
# SUBSTITUTE the path: replace ROUTE_PATH below with a real route from the app model
# e.g. agent-browser open "$BASE_URL/settings"
agent-browser open "$BASE_URL/ROUTE_PATH"
agent-browser wait --load networkidle
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  ariaCurrent: document.querySelector('[aria-current="page"]')?.textContent?.trim(),
  activeByClass: Array.from(document.querySelectorAll('nav a, [role="navigation"] a, [role="menuitem"]')).find(a =>
    /active|selected|current/.test(a.className)
  )?.textContent?.trim(),
  // For hash-routed apps, pathname is always "/" — check hash instead
  currentPath: window.location.hash || window.location.pathname
})
EVALEOF
```

Expected: active state still correct even though navigation was via URL, not a click.

If neither `ariaCurrent` nor `activeByClass` is set on any link: the app may not implement active states, or may use a mechanism not detectable by class or aria attribute (e.g. inline style or SVG fill). Take a screenshot and note it as a suspected Medium finding — active state not reliably detectable. **PASS / FAIL / SUSPECTED**

---

## Tab groups: clicking tabs switches content

```bash
# Run the preflight in ../session-preflight.md first.
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify(Array.from(document.querySelectorAll('[role="tab"]')).map(t => ({
  text: t.textContent?.trim(),
  selected: t.getAttribute('aria-selected')
})))
EVALEOF
```

For each tab — re-snapshot, click it, assert the active tab changed and panel loaded:

```bash
agent-browser snapshot -i -C
agent-browser click @eN
agent-browser wait 800
agent-browser screenshot --full
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  activeTab: document.querySelector('[aria-selected="true"]')?.textContent?.trim(),
  panelVisible: !!(document.querySelector('[role="tabpanel"]:not([hidden])')),
  panelNotEmpty: (document.querySelector('[role="tabpanel"]:not([hidden])')?.children?.length ?? 0) > 0,
  panelText: document.querySelector('[role="tabpanel"]:not([hidden])')?.textContent?.trim()?.substring(0, 80)
})
EVALEOF
```

Expected: `aria-selected="true"` moved to clicked tab, `panelVisible` true, `panelNotEmpty` true. If `panelNotEmpty` is false but `panelVisible` is true: wait an extra 1000ms and re-check — panel may be lazy-loading. If still empty after retry, that's a High finding (tab content failed to load). **PASS / FAIL**

---

## Logo / home link returns to root

```bash
agent-browser snapshot -i -C
# Identify logo or home link ref (usually top-left of header)
agent-browser click @eN
agent-browser wait --load networkidle
agent-browser get url
```

Expected: the URL resolves to `/`, the app's home route, or the hash-based home route. The logo should never navigate to a 404 or a deeply nested route. **PASS / FAIL**

---

## Breadcrumbs: parent links navigate correctly

```bash
# Run the preflight in ../session-preflight.md first.
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify(Array.from(document.querySelectorAll('[aria-label*="breadcrumb" i] a, [class*="breadcrumb"] a')).map(a => ({
  text: a.textContent?.trim(),
  href: a.getAttribute('href')
})))
EVALEOF
```

If breadcrumbs found — click a parent item (not the last/current one):

```bash
agent-browser snapshot -i -C
agent-browser click @eN
agent-browser wait --load networkidle
agent-browser get url
```

Expected: URL matches breadcrumb href. **PASS / FAIL**

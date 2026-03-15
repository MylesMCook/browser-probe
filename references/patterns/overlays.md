# Overlays Patterns

---

Before using this file:

- Take a fresh `agent-browser snapshot -i -C`.
- Run the preflight in [../session-preflight.md](../session-preflight.md) before any `eval`.
- Prefer direct commands when they answer the question, but keep `eval` for overlay structure and focus checks.

## Modal — opens, has close mechanisms, dismisses cleanly

First find a button likely to open a modal — look for common trigger patterns:

```bash
# Run the preflight in ../session-preflight.md first.
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify(Array.from(document.querySelectorAll('button, [role="button"]')).filter(b => {
  const text = (b.textContent || '').trim().toLowerCase();
  const label = (b.getAttribute('aria-label') || '').toLowerCase();
  const combined = text + ' ' + label;
  // Common modal triggers
  return /new|create|add|edit|open|view|details|settings|configure|invite|upload|import/i.test(combined)
    && !/(close|cancel|dismiss)/i.test(combined);
}).slice(0, 5).map(b => ({
  text: b.textContent?.trim(),
  label: b.getAttribute('aria-label'),
  class: b.className?.substring(0, 50)
})))
EVALEOF
```

If no obvious trigger buttons: check for icon buttons that might open overlays, or look at the snapshot for button refs near `+`, pencil, or gear icons.

Trigger the modal:

```bash
agent-browser snapshot -i -C
# Identify trigger button ref from the list above or the snapshot
agent-browser click @eN
agent-browser wait 500
agent-browser screenshot --full
```

Assert modal opened correctly:

```bash
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  hasModal: !!(
    document.querySelector('[role="dialog"]') ||
    document.querySelector('[data-state="open"][class*="modal" i]') ||
    document.querySelector('[data-state="open"][class*="dialog" i]') ||
    document.querySelector('[class*="modal" i][style*="display: block"]') ||
    document.querySelector('[data-radix-dialog-content]') ||
    document.querySelector('[class*="DialogContent"], [class*="ModalContent"]')
  ),
  hasBackdrop: !!(
    document.querySelector('[class*="overlay"], [class*="backdrop"]') ||
    document.querySelector('[data-radix-dialog-overlay]') ||
    document.querySelector('[class*="Overlay"], [class*="Backdrop"]')
  ),
  heading: (
    document.querySelector('[role="dialog"] h2, [role="dialog"] h3') ||
    document.querySelector('[data-radix-dialog-content] h2, [data-radix-dialog-content] h3') ||
    document.querySelector('[class*="DialogContent"] h2, [class*="DialogContent"] h3')
  )?.textContent?.trim(),
  hasCloseButton: !!(
    document.querySelector('[aria-label*="close" i], [class*="close"], [class*="dismiss"]') ||
    document.querySelector('[data-radix-dialog-close]') ||
    document.querySelector('button[class*="Close"], button[class*="cancel" i]')
  )
})
EVALEOF
```

Expected: `hasModal` true, heading non-empty, `hasCloseButton` true. **PASS / FAIL**

**Close via X button:**

```bash
agent-browser snapshot -i -C
agent-browser click @eN
agent-browser wait 300
agent-browser eval --stdin <<'EVALEOF'
!(
  document.querySelector('[role="dialog"]') ||
  document.querySelector('[data-state="open"][class*="dialog" i]') ||
  document.querySelector('[data-radix-dialog-content]') ||
  document.querySelector('[class*="DialogContent"]:not([hidden])')
)
EVALEOF
```

Expected: `true`. **PASS / FAIL**

**Close via Escape key** (re-open first):

```bash
agent-browser snapshot -i -C
agent-browser click @eN
agent-browser wait 500
agent-browser press Escape
agent-browser wait 300
agent-browser eval --stdin <<'EVALEOF'
!(
  document.querySelector('[role="dialog"]') ||
  document.querySelector('[data-state="open"][class*="dialog" i]') ||
  document.querySelector('[data-radix-dialog-content]') ||
  document.querySelector('[class*="DialogContent"]:not([hidden])')
)
EVALEOF
```

Expected: `true`. Some apps intentionally block Escape on critical dialogs — note if so, not a bug. **PASS / FAIL**

**Hypothesis check — focus trap:**

```bash
agent-browser snapshot -i -C
# Re-open modal
agent-browser click @eN
agent-browser wait 500
agent-browser eval --stdin <<'EVALEOF'
// Use the widest possible dialog selector to handle Radix, MUI, vanilla
const dialogEl =
  document.querySelector('[role="dialog"]') ||
  document.querySelector('[data-radix-dialog-content]') ||
  document.querySelector('[class*="DialogContent"], [class*="ModalContent"]');
JSON.stringify({
  foundDialog: !!dialogEl,
  focusableInsideCount: dialogEl?.querySelectorAll('a, button, input, textarea, select, [tabindex]:not([tabindex="-1"])').length ?? 0,
  focusableOutsideCount: Array.from(document.querySelectorAll('a, button, input')).filter(el =>
    !el.closest('[role="dialog"], [data-radix-dialog-content], [class*="DialogContent"]')
  ).length
})
EVALEOF
```

If `focusableOutsideCount` > 0 and `focusableInsideCount` > 0, press Tab several times and check focus doesn't escape:

```bash
agent-browser press Tab
agent-browser press Tab
agent-browser press Tab
agent-browser press Tab
agent-browser press Tab
agent-browser eval --stdin <<'EVALEOF'
const isInsideDialog = !!document.activeElement?.closest(
  '[role="dialog"], [data-radix-dialog-content], [class*="DialogContent"]'
);
isInsideDialog
EVALEOF
```

Expected: `true` — focus stays inside dialog. If `foundDialog` was `false`, skip this test and note the dialog selector was not matched. If focus escapes to page content behind the backdrop, that's a Medium accessibility finding. **PASS / FAIL / SKIPPED**

---

## Confirmation dialog — cancel and confirm both work

First find a destructive action trigger to click:

```bash
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify(Array.from(document.querySelectorAll('button, [role="button"], a')).filter(el => {
  const text = (el.textContent || '').trim();
  const label = el.getAttribute('aria-label') || '';
  return /delete|remove|reset|destroy|archive|deactivate|löschen|supprimer|eliminar/i.test(text + ' ' + label);
}).slice(0, 3).map(el => ({ text: el.textContent?.trim(), label: el.getAttribute('aria-label') })))
EVALEOF
```

If no destructive buttons are visible on the current page: navigate to a page with a list of records (users, items, etc.) where a delete action is likely to exist. If no destructive action exists anywhere in the app, skip this section entirely.

Trigger the destructive action. First capture the baseline row count so the cancel assertion has a number to compare against:

```bash
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  baselineRowCount: document.querySelectorAll('table tbody tr, [class*="card"], [class*="item"]').length
})
EVALEOF
```

```bash
agent-browser snapshot -i -C
agent-browser click @eN
agent-browser wait 500
agent-browser screenshot --full
```

Assert confirmation dialog appeared:

```bash
agent-browser eval --stdin <<'EVALEOF'
// Check for dialog presence by role — don't rely on button text labels (they vary by language)
const dialog = document.querySelector('[role="alertdialog"], [role="dialog"]');
const allButtons = dialog ? Array.from(dialog.querySelectorAll('button')) : [];
JSON.stringify({
  hasDialog: !!dialog,
  dialogHeading: dialog?.querySelector('h2, h3, [role="heading"]')?.textContent?.trim(),
  buttonCount: allButtons.length,
  buttonLabels: allButtons.map(b => b.textContent?.trim()).filter(Boolean)
})
EVALEOF
```

Expected: dialog present, `buttonCount` >= 2 (at minimum a confirm and a cancel action), `buttonLabels` non-empty. Use the returned labels to identify which button is confirm and which is cancel in the steps below — do not assume English labels. **PASS / FAIL**

Test cancel path:

```bash
agent-browser snapshot -i -C
# Identify Cancel button ref from the buttonLabels returned above
agent-browser click @eN
agent-browser wait 300
agent-browser eval --stdin <<'EVALEOF'
// Check both roles — apps use either for confirmation dialogs
!(document.querySelector('[role="alertdialog"]') || document.querySelector('[role="dialog"]'))
EVALEOF
```

Expected: `true` (dialog gone), action NOT performed. Verify the item was not deleted — check it still appears in the list:

```bash
agent-browser eval --stdin <<'EVALEOF'
// The item that was targeted for deletion should still be in the DOM.
// Compare against the baselineRowCount captured before triggering the action.
JSON.stringify({
  rowCount: document.querySelectorAll('table tbody tr, [class*="card"], [class*="item"]').length,
  // If you noted the item's text before triggering, check it's still present:
  // itemStillPresent: document.body.textContent.includes('NOTED_ITEM_TEXT')
})
EVALEOF
```

Expected: `rowCount` equals the baseline captured before the destructive action was triggered. **PASS / FAIL**

Test confirm path — re-trigger the destructive action:

```bash
agent-browser snapshot -i -C
agent-browser click @eN
agent-browser wait 500
agent-browser snapshot -i -C
# Identify Confirm/Delete/Yes button ref
agent-browser click @eN
agent-browser wait --load networkidle
agent-browser screenshot --full
```

Expected: dialog gone, action WAS performed (item deleted, record reset, etc.), and a success signal (toast, redirect, or updated list) confirms it. If the confirm button does nothing visible, that's a High finding. **PASS / FAIL**

---

## Toast / snackbar — appears and auto-dismisses

After an action that should produce a toast (save, copy, delete):

```bash
agent-browser wait 1000
agent-browser screenshot --full
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  hasToast: !!(
    document.querySelector('[class*="toast"], [class*="Toast"]') ||
    document.querySelector('[class*="snackbar"], [class*="Snackbar"]') ||
    document.querySelector('[role="status"], [role="alert"]') ||
    document.querySelector('[data-sonner-toast]') ||
    document.querySelector('[class*="notification" i]')
  ),
  toastText: (
    document.querySelector('[data-sonner-toast]') ||
    document.querySelector('[class*="toast"], [class*="Toast"]') ||
    document.querySelector('[role="status"]')
  )?.textContent?.trim()?.substring(0, 100)
})
EVALEOF
```

Expected: `hasToast` true, `toastText` is a meaningful message (not empty, not raw JSON). **PASS / FAIL**

Assert auto-dismisses after a reasonable timeout:

```bash
agent-browser wait 6000
agent-browser eval '!document.querySelector("[class*=\"toast\"], [class*=\"snackbar\"]")'
```

Expected: `true`. If toasts stack infinitely without dismissing, that's a Medium finding. **PASS / FAIL**

---

## Drawer / side panel — opens and closes

First find a trigger likely to open a drawer or side panel:

```bash
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify(Array.from(document.querySelectorAll('button, [role="button"], a')).filter(b => {
  const text = (b.textContent || '').trim().toLowerCase();
  const label = (b.getAttribute('aria-label') || '').toLowerCase();
  const cls = (b.className || '').toLowerCase();
  const combined = text + ' ' + label + ' ' + cls;
  return /detail|panel|drawer|slide|info|expand|open|filter|sidebar/i.test(combined)
    && !/(close|cancel|dismiss|delete|remove)/i.test(combined);
}).slice(0, 5).map(b => ({
  text: b.textContent?.trim(),
  label: b.getAttribute('aria-label'),
  class: b.className?.substring(0, 50)
})))
EVALEOF
```

If no obvious trigger: look in the snapshot for row-level or item-level buttons near the right side of a list or table — drawers often open from clicking a row or an info icon. If no drawer trigger exists anywhere in the app, skip this section.

```bash
agent-browser snapshot -i -C
# Identify drawer trigger ref from the list above or the snapshot
agent-browser click @eN
agent-browser wait 500
agent-browser screenshot --full
```

Assert drawer opened:

```bash
agent-browser eval --stdin <<'EVALEOF'
const drawerEl =
  document.querySelector('[class*="drawer" i], [class*="side-panel" i], [class*="sheet" i]') ||
  document.querySelector('[vaul-drawer]') ||
  document.querySelector('[data-state="open"][class*="Sheet" i]') ||
  document.querySelector('[class*="MuiDrawer-root"]');
JSON.stringify({
  hasDrawer: !!drawerEl,
  drawerVisible: (() => {
    if (!drawerEl) return false;
    const style = window.getComputedStyle(drawerEl);
    return style.display !== 'none' && style.visibility !== 'hidden' && style.opacity !== '0';
  })()
})
EVALEOF
```

Expected: `hasDrawer` true, `drawerVisible` true. **PASS / FAIL**

Close via Escape:

```bash
agent-browser press Escape
agent-browser wait 300
agent-browser screenshot --full
agent-browser eval --stdin <<'EVALEOF'
const drawerEl =
  document.querySelector('[class*="drawer" i], [class*="side-panel" i], [class*="sheet" i]') ||
  document.querySelector('[vaul-drawer]') ||
  document.querySelector('[class*="MuiDrawer-root"]');
// Drawer gone, or still present but hidden
const drawerGone = !drawerEl || (() => {
  const s = window.getComputedStyle(drawerEl);
  return s.display === 'none' || s.visibility === 'hidden' || s.opacity === '0';
})();
drawerGone
EVALEOF
```

Expected: `true` — drawer dismissed. If still visible, that's a Medium finding (Escape key doesn't close drawer). **PASS / FAIL**

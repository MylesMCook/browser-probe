# Forms Patterns

---

Before using this file:

- Take a fresh `agent-browser snapshot -i -C`.
- Run the preflight in [../session-preflight.md](../session-preflight.md) before any `eval`.
- Prefer direct `get count`, `get text`, and `get url` checks when they are enough.

## Discover form structure before testing

```bash
# Run the preflight in ../session-preflight.md first.
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  nativeInputs: Array.from(document.querySelectorAll('input, textarea, select')).map(el => ({
    tag: el.tagName, type: el.type || null, name: el.name || null,
    placeholder: el.placeholder || null, required: el.required
  })),
  customInputs: Array.from(document.querySelectorAll('[role="combobox"], [role="switch"], [role="checkbox"], [role="radio"], [contenteditable="true"]')).map(el => ({
    role: el.getAttribute('role'),
    label: el.getAttribute('aria-label') || el.closest('label')?.textContent?.trim()
  })),
  submitButton: document.querySelector('button[type="submit"]')?.textContent?.trim() ||
    Array.from(document.querySelectorAll('button')).find(b => /submit|save|create|add|update/i.test(b.textContent))?.textContent?.trim(),
  // Detect component-library selects that render their options portalled to document.body
  // These cannot be filled with `select @ref` — they need click → wait → click option
  hasPortalledSelects: !!(
    document.querySelector('[class*="MuiSelect"], [class*="ant-select"], [class*="Select__control"]') ||
    document.querySelector('[role="combobox"][aria-haspopup="listbox"]')
  )
})
EVALEOF
```

**If `hasPortalledSelects` is true:** do not use `agent-browser select @ref` for these controls — it only works with native `<select>` elements. Instead: snapshot, click the combobox trigger ref, wait 300ms, re-snapshot to get the option refs that appeared, then click the desired option ref.

---

## Happy path — fill all required fields and submit

Before filling, discover what options are available for any select inputs:

```bash
# Run the preflight in ../session-preflight.md first.
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify(Array.from(document.querySelectorAll('select')).map(s => ({
  name: s.name || s.id,
  options: Array.from(s.options).map(o => o.text).filter(t => t && t.trim())
})))
EVALEOF
```

Use the first non-empty, non-placeholder option value when filling selects below.

```bash
agent-browser snapshot -i -C
# Fill each required field — identify refs from snapshot output
# Text inputs:
agent-browser fill @eN "Sample value"
# Textareas:
agent-browser fill @eN "Longer description text"
# Native selects — SUBSTITUTE the actual option text from the discovery eval above:
# e.g. if discovery returned ["Active", "Inactive"], use "Active" not the placeholder text
agent-browser select @eN "SUBSTITUTE_first_real_option_HERE"
# Checkboxes:
agent-browser check @eN
# Submit:
agent-browser click @eN
agent-browser wait --load networkidle
agent-browser screenshot --full
```

Assert success — toast/confirmation appeared or form dismissed:

```bash
agent-browser eval --stdin <<'EVALEOF'
const dialogGone = !document.querySelector('[role="dialog"]');
const formGone = !document.querySelector('form input');
JSON.stringify({
  hasSuccessSignal: !!document.querySelector('[class*="success" i], [class*="toast"], [role="status"]'),
  successText: document.querySelector('[class*="success" i], [class*="toast"], [role="status"]')?.textContent?.trim()?.substring(0, 100),
  // Form dismissed = dialog gone AND no visible form inputs (both must be true)
  formDismissed: dialogGone && formGone
})
EVALEOF
```

Expected: success signal visible OR form dismissed. If neither, that's a High finding — silent failure. **PASS / FAIL**

---

## Required fields — submit without filling anything

Navigate fresh to the form — use the route from the app model where this form lives. If the form is in a modal, navigate to the page that contains the modal trigger and re-open it. SUBSTITUTE the actual path before executing:

```bash
# SUBSTITUTE: replace FORM_ROUTE below with the actual path from the app model
# e.g. agent-browser open "$BASE_URL/users/new"  or  "$BASE_URL/settings"
agent-browser open "$BASE_URL/FORM_ROUTE"
agent-browser wait --load networkidle
agent-browser snapshot -i -C
# Click submit with nothing filled
agent-browser click @eN
agent-browser wait 1000
agent-browser screenshot --full
```

Assert validation appeared:

```bash
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  errors: Array.from(document.querySelectorAll('[class*="error" i], [aria-invalid="true"], [class*="invalid" i]')).map(el => el.textContent?.trim()).filter(Boolean),
  formStillPresent: !!document.querySelector('form, [role="dialog"]')
})
EVALEOF
```

Expected: at least one error, form not submitted. **PASS / FAIL**

Hypothesis check — do validation errors clear after the user corrects the field?

```bash
agent-browser snapshot -i -C
# Identify the first errored input ref
agent-browser fill @eN "corrected value"
agent-browser wait 500
agent-browser get count '[aria-invalid="true"]'
```

Expected: error cleared from that field (count decreased). If errors persist after correction, that's a Medium finding. **PASS / FAIL**

---

## Double-submit prevention

**All required fields must be filled before clicking submit.** If any required field is empty, client-side validation blocks the first click and both clicks get rejected before reaching the network — the test never exercises double-submit. Use the same values that worked in the happy path above.

```bash
agent-browser snapshot -i -C
# Fill EVERY required field the form has — not just the first one.
# Use the same values from the happy path. If you skip any required field,
# validation will block both clicks and this test is meaningless.
agent-browser fill @eN "test value"
# Fill remaining required fields:
agent-browser fill @eN "test value 2"
# ... continue for all required inputs, selects, and checkboxes

# Once all required fields are filled, rapidly click submit twice:
agent-browser snapshot -i -C
agent-browser click @eN
agent-browser click @eN
agent-browser wait --load networkidle
agent-browser screenshot --full
```

Assert only one submission occurred (no duplicate records, no double toast):

```bash
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  toastCount: document.querySelectorAll('[class*="toast"], [role="status"]').length,
  submitButtonDisabled: document.querySelector('button[type="submit"], button[disabled]')?.disabled
})
EVALEOF
```

Expected: one toast/confirmation, submit button disabled during submission. If `toastCount` > 1 or duplicates visible, that's a High finding. **PASS / FAIL**

---

## Cancel / discard — data is not saved

```bash
agent-browser snapshot -i -C
# Fill a field with recognizable data
agent-browser fill @eN "data-i-will-discard-12345"
# Find and click Cancel/Close button ref
agent-browser click @eN
agent-browser wait 500
agent-browser screenshot --full
```

Assert form dismissed and test data is gone:

```bash
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  dialogGone: !document.querySelector('[role="dialog"]'),
  url: window.location.href,
  testDataVisible: document.body.textContent.includes('data-i-will-discard-12345')
})
EVALEOF
```

Expected:
- If the form was in a dialog: `dialogGone` true and `testDataVisible` false.
- If the form was on its own page: `url` changed (navigated away) and `testDataVisible` false.
- In either case: if the string appears in the page, data was saved despite cancelling — that's a High finding.

**PASS / FAIL**

---

## Edit existing record — value persists after save

Navigate to a record detail page with an editable field:

```bash
agent-browser snapshot -i -C
# Identify edit trigger ref (Edit button, pencil icon, or inline click)
agent-browser click @eN
agent-browser wait 500
agent-browser snapshot -i -C
# Note the original value, then overwrite with recognizable test value
agent-browser fill @eN "probe-updated-value-99"
agent-browser snapshot -i -C
# Save
agent-browser click @eN
agent-browser wait --load networkidle
agent-browser screenshot --full
```

Assert updated value persists — reload the page first so the input field value can't cause a false pass:

```bash
agent-browser get url
# ↑ Copy the URL printed above, then paste it into the open command below:
agent-browser open "PASTE_CAPTURED_URL_HERE"
agent-browser wait --load networkidle
agent-browser eval 'document.body.textContent.includes("probe-updated-value-99")'
```

Expected: `true` — the value survived a reload, confirming it was actually saved and not just held in component state. **PASS / FAIL**

---

## Long input — does not break layout

```bash
agent-browser snapshot -i -C
# Find any text input ref
agent-browser fill @eN "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"
agent-browser screenshot --full
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  inputOverflows: (() => {
    const input = document.querySelector('input[type="text"], textarea');
    if (!input) return null;
    return input.scrollWidth > input.clientWidth + 10;
  })(),
  layoutBreaks: document.body.scrollWidth > window.innerWidth + 20
})
EVALEOF
```

Expected: `layoutBreaks` false. Input overflow is fine (that's expected), but the surrounding layout should not break. **PASS / FAIL**

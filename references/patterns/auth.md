# Auth Patterns

---

## Discover form structure before testing

Navigate to the app root first — the app will redirect to login if auth is required. Do not assume the browser is already on the login page.

```bash
agent-browser open "$BASE_URL"
agent-browser wait --load networkidle
agent-browser snapshot -i -C
```

```bash
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  inputs: Array.from(document.querySelectorAll('input')).map(i => ({
    type: i.type, name: i.name, placeholder: i.placeholder, required: i.required
  })),
  submitButton: document.querySelector('button[type="submit"], button')?.textContent?.trim(),
  hasOAuth: Array.from(document.querySelectorAll('button, a')).filter(el =>
    /google|github|microsoft|okta|sso/i.test(el.textContent + el.className)
  ).map(el => el.textContent?.trim())
})
EVALEOF
```

---

## Before testing auth — establish credentials

Check whether the user has provided credentials:

- Look in the conversation for any email/username and password mentioned.
- Check common env vars: `TEST_EMAIL`, `TEST_USER`, `TEST_PASSWORD`, `CYPRESS_USERNAME`, `CYPRESS_PASSWORD`, etc.:

```bash
echo "EMAIL: ${TEST_EMAIL:-${CYPRESS_USERNAME:-not set}}"
echo "PASSWORD: ${TEST_PASSWORD:-${CYPRESS_PASSWORD:-not set}}"
```

If credentials are available: use them in the tests below, replacing `test@example.com` / `password123`.

If credentials are not available: note this prominently in the report. Skip the happy path and session tests. Run only: empty submission, protected route redirect (using the base URL), and OAuth discovery. Do NOT mark the happy path as "FAIL" — it was not tested.

---

## Invalid credentials — error handling

**Run this test BEFORE the happy path.** Some apps lock accounts or show CAPTCHAs after 3–5 failed attempts. Testing invalid credentials first avoids locking the valid account used in the happy path test. If no valid credentials are available, order doesn't matter.

```bash
agent-browser open "$BASE_URL"
agent-browser wait --load networkidle
agent-browser snapshot -i -C
agent-browser fill @eN "wrong@example.com"
agent-browser fill @eN "wrongpassword"
agent-browser click @eN
agent-browser wait 2000
agent-browser screenshot --full
```

Assert error shown, form stays on login page:

```bash
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  hasError: !!document.querySelector('[class*="error" i], [role="alert"], [class*="invalid" i]'),
  errorText: document.querySelector('[class*="error" i], [role="alert"]')?.textContent?.trim()?.substring(0, 100),
  stillOnLogin: !!document.querySelector('input[type="password"]')
})
EVALEOF
```

Expected: `hasError` true, `errorText` is a meaningful message, `stillOnLogin` true. **PASS / FAIL**

Hypothesis check — does the error clear when the user starts typing again?

```bash
agent-browser snapshot -i -C
# Identify email input ref
agent-browser fill @eN "corrected@example.com"
agent-browser wait 300
agent-browser eval '!!document.querySelector("[class*=\"error\" i], [role=\"alert\"]")'
```

Expected: error clears on new input (false). If it stays visible while typing valid credentials, that's a Medium finding. **PASS / FAIL**

---

## Empty submission — required field validation

**Run this test BEFORE the happy path** — it must execute while the user is not yet logged in. After a successful login the app won't show the login form, making this test impossible.

Navigate to the login page. `$BASE_URL` will redirect to login if the app requires auth:

```bash
agent-browser open "$BASE_URL"
agent-browser wait --load networkidle
agent-browser snapshot -i -C
# Click submit without filling anything
agent-browser click @eN
agent-browser wait 1000
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  validationMessages: Array.from(document.querySelectorAll('[class*="error" i], :invalid, [aria-invalid="true"]')).map(el => el.textContent?.trim() || el.validationMessage).filter(Boolean),
  formStillPresent: !!document.querySelector('input[type="password"]')
})
EVALEOF
```

Expected: validation errors or browser native validation visible, form did not submit. **PASS / FAIL**

---

## Valid credentials — happy path

**Substitute real credentials before running this.** The fill commands below contain placeholders — the agent must replace them with actual values from the "Before testing auth" section above before executing. Do not execute them with the placeholder text in place.

```bash
agent-browser snapshot -i -C
# Identify email/username input ref from snapshot, e.g. @e2 [input placeholder="Email"]
# REPLACE THE FILL VALUE: use the actual email/username from credentials, not the placeholder text
agent-browser fill @eN "SUBSTITUTE_EMAIL_HERE"
# Identify password input ref
# REPLACE THE FILL VALUE: use the actual password from credentials
agent-browser fill @eN "SUBSTITUTE_PASSWORD_HERE"
# Identify submit button ref
agent-browser click @eN
agent-browser wait --load networkidle
agent-browser screenshot --full
```

Assert success — redirected away from login, authenticated UI visible:

```bash
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  url: window.location.href,
  loginFormGone: !document.querySelector('input[type="password"]'),
  hasUserIndicator: !!document.querySelector('[class*="avatar"], [class*="user-menu"], [aria-label*="account" i], [aria-label*="profile" i]')
})
EVALEOF
```

Expected: URL changed away from login, `loginFormGone` true, user indicator present. **PASS / FAIL**

---

## Protected route redirect when unauthenticated

Choose a protected route to test: use a non-root route from the app model that is not the login page itself. If the app model lists routes like `/dashboard`, `/settings`, or `/users`, use one of those. If no routes were discovered (because auth blocked discovery), try `$BASE_URL/dashboard`, `$BASE_URL/app`, or `$BASE_URL/admin` as common defaults.

Navigate directly to that URL without logging in. SUBSTITUTE the actual path from the app model before executing (e.g. `/dashboard`, `/settings`, `/users`):

```bash
# SUBSTITUTE: replace ROUTE_PATH with the actual protected path from the app model
# e.g. agent-browser open "$BASE_URL/dashboard"
agent-browser open "$BASE_URL/ROUTE_PATH"
agent-browser wait --load networkidle
agent-browser eval 'JSON.stringify({ url: window.location.href, hasLoginForm: !!document.querySelector("input[type=password]") })'
```

Expected: redirected to login page (`hasLoginForm` true), not shown the protected content. If the protected content is briefly visible before redirect (flash), that's a Medium finding. **PASS / FAIL**

---

## Session persists after page reload

After a successful login, capture the current URL then navigate to it fresh (simulating a reload):

```bash
agent-browser eval 'window.location.href'
# ↑ Copy the URL printed above — it is the authenticated page you're on.
# Paste it into the open command below, replacing PASTE_CAPTURED_URL_HERE:
agent-browser open "PASTE_CAPTURED_URL_HERE"
agent-browser wait --load networkidle
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  stillAuthenticated: !document.querySelector('input[type="password"]'),
  url: window.location.href
})
EVALEOF
```

Expected: still authenticated, not redirected to login. If session is stored in memory only (not cookie/localStorage), reload will log the user out — that's a High finding. **PASS / FAIL**

---

## Logout works and clears session

First check whether logout is accessible via a dropdown menu or a direct link:

```bash
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  hasDirectLogout: !!Array.from(document.querySelectorAll('a, button')).find(el =>
    /log.?out|sign.?out/i.test(el.textContent + el.getAttribute('aria-label') || '')
  ),
  hasUserMenuTrigger: !!(
    document.querySelector('[class*="avatar"], [class*="user-menu"], [aria-label*="account" i], [aria-label*="profile" i]') ||
    document.querySelector('[class*="UserMenu"], [class*="AccountMenu"]')
  )
})
EVALEOF
```

**If `hasDirectLogout` is true:** find and click the logout link directly:

```bash
agent-browser snapshot -i -C
# Identify the logout link/button ref — look for text matching "log out", "sign out", or similar
agent-browser click @eN
agent-browser wait --load networkidle
agent-browser screenshot --full
```

**If `hasUserMenuTrigger` is true and no direct logout:** open the user menu first:

```bash
agent-browser snapshot -i -C
# Identify user menu / avatar trigger ref
agent-browser click @eN
agent-browser wait 300
agent-browser snapshot -i -C
# Identify logout button in the opened menu
agent-browser click @eN
agent-browser wait --load networkidle
agent-browser screenshot --full
```

Assert logged out:

```bash
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  url: window.location.href,
  hasLoginForm: !!document.querySelector('input[type="password"]')
})
EVALEOF
```

Expected: redirected to login or root, login form visible. **PASS / FAIL**

Verify session is actually cleared (not just redirected). Use the same protected route path tested earlier. SUBSTITUTE the same `ROUTE_PATH` used in the "Protected route redirect" section:

```bash
# SUBSTITUTE: same ROUTE_PATH used in the protected route test above
agent-browser open "$BASE_URL/ROUTE_PATH"
agent-browser wait --load networkidle
agent-browser get url
```

Expected: redirected to login, not shown the protected content. **PASS / FAIL**

---

## OAuth buttons present and reachable

If OAuth buttons were found in discovery:

```bash
agent-browser snapshot -i -C
# Identify an OAuth button ref (Google, GitHub, etc.)
agent-browser click @eN
agent-browser wait 2000
agent-browser get url
```

**Note:** OAuth often opens in a new tab. If the URL here is unchanged and still on the login page, the provider opened a new tab — consult `recovery.md` → "The app opened a new tab" for how to handle this. The URL will be the OAuth provider's domain (accounts.google.com, github.com, etc.) if it stayed in the same tab. **PASS / FAIL / PARTIAL**

Navigate back — use `$BASE_URL` directly; the app will redirect to its login page:

```bash
agent-browser open "$BASE_URL"
agent-browser wait --load networkidle
```

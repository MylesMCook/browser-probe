# Stage 1 - Model the app

Do not start poking randomly. Build a concise model first.

## 1. Start the session and open the target

```bash
export AGENT_BROWSER_SESSION="${AGENT_BROWSER_SESSION:?unset}"
export AGENT_BROWSER_STATE_FILE="${AGENT_BROWSER_STATE_FILE:-/tmp/${AGENT_BROWSER_SESSION}-auth-state.json}"
agent-browser open "$TARGET_URL"
agent-browser wait --load networkidle
agent-browser snapshot -i -C
```

If the target is not localhost-style, add `--allowed-domains` with the target host before opening it.

Before every later `eval` block in Stage 1, run the preflight in [session-preflight.md](session-preflight.md). Keep each shell block self-contained instead of assuming the previous block's JS context is still live.

## 2. Identify the scope

Decide whether you are modeling:
- the whole app
- one named feature
- one specific route or flow

Do not map the whole product if the user clearly scoped the request to auth, checkout, billing, or a single feature.

## 3. Record the basics

Capture:
- current URL
- app purpose in one sentence
- auth status: not required, logged in, or blocked
- major routes or major in-scope views
- component types present
- likely critical user flows

Useful checks:

```bash
export AGENT_BROWSER_SESSION="${AGENT_BROWSER_SESSION:?unset}"
# Run the preflight in references/session-preflight.md first.
agent-browser get url
agent-browser get title
agent-browser eval 'JSON.stringify({ pathname: window.location.pathname, hash: window.location.hash, lang: document.documentElement.lang || navigator.language || "unknown" })'
```

## 4. Map routes or views

Start with visible navigation and obvious calls to action.

Useful route-discovery snippet:

```bash
export AGENT_BROWSER_SESSION="${AGENT_BROWSER_SESSION:?unset}"
# Run the preflight in references/session-preflight.md first.
agent-browser eval --stdin <<'EOF'
JSON.stringify(
  Array.from(document.querySelectorAll('a[href]'))
    .map((a) => ({ text: a.textContent?.trim(), href: a.getAttribute('href') }))
    .filter((a) => a.text && a.href && !a.href.startsWith('mailto:') && !a.href.startsWith('tel:'))
    .slice(0, 30)
)
EOF
```

Mode guidance:
- `smoke`: map only the top-level path or named feature entry point
- `balanced`: map the main routes or main in-scope views
- `bug-hunt`: also map obvious secondary paths, deep links, and responsive-only navigation if relevant

## 5. Inventory component types

Use a quick structural read:

```bash
export AGENT_BROWSER_SESSION="${AGENT_BROWSER_SESSION:?unset}"
# Run the preflight in references/session-preflight.md first.
agent-browser eval --stdin <<'EOF'
JSON.stringify({
  hasNav: !!document.querySelector('nav, [role="navigation"]'),
  hasTabs: !!document.querySelector('[role="tab"], [role="tablist"]'),
  formCount: document.querySelectorAll('form, input, textarea, select').length,
  tableCount: document.querySelectorAll('table, [role="table"], [role="grid"]').length,
  dialogCount: document.querySelectorAll('[role="dialog"], [data-radix-dialog-content]').length,
  searchCount: document.querySelectorAll('input[type="search"], [role="searchbox"]').length,
  passwordInputs: document.querySelectorAll('input[type="password"]').length
})
EOF
```

Note only the component families that will matter in Stage 2.

## 6. Write the model

Before leaving Stage 1, write this down:

```text
APP MODEL
Mode: [smoke | balanced | bug-hunt]
Scope: [full app | specific feature]
Purpose: [one sentence]
Auth status: [not required | logged in | blocked]
Routing: [path-based | hash-based | unknown]
Locale: [en-US | de-DE | unknown]

Routes or views discovered:
- [route or view] -> [what is here]

Critical or in-scope flows:
- [flow]
- [flow]

Component inventory:
- navigation: [...]
- forms: [...]
- data display: [...]
- overlays: [...]
- search/filter: [...]
```

Keep it concise. Stage 2 depends on it.

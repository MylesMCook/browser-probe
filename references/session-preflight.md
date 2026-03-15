# Session Preflight

Run this before any shell block that depends on `eval` or structured DOM reads.

`AGENT_BROWSER_SESSION` is live isolation, not a promise that the previous JS execution context is still valid in a later shell invocation.

## Bootstrap once per run

```bash
export AGENT_BROWSER_SESSION="${AGENT_BROWSER_SESSION:?unset}"
export TARGET_URL="${TARGET_URL:?unset}"
export BASE_URL="${BASE_URL:-$TARGET_URL}"
export TARGET_HOST="${TARGET_HOST:-}"
export AGENT_BROWSER_STATE_FILE="${AGENT_BROWSER_STATE_FILE:-/tmp/${AGENT_BROWSER_SESSION}-auth-state.json}"
```

After a successful login or other expensive authenticated setup, checkpoint state explicitly:

```bash
export AGENT_BROWSER_SESSION="${AGENT_BROWSER_SESSION:?unset}"
export AGENT_BROWSER_STATE_FILE="${AGENT_BROWSER_STATE_FILE:-/tmp/${AGENT_BROWSER_SESSION}-auth-state.json}"
agent-browser state save "$AGENT_BROWSER_STATE_FILE"
```

## Preflight before `eval`

Capture the current page state, then run a tiny sanity eval:

```bash
export AGENT_BROWSER_SESSION="${AGENT_BROWSER_SESSION:?unset}"
CURRENT_URL="$(agent-browser get url 2>/dev/null || true)"
[ -n "$CURRENT_URL" ] && [ "$CURRENT_URL" != "about:blank" ] || CURRENT_URL="$TARGET_URL"
agent-browser snapshot -i -C
agent-browser eval 'JSON.stringify({ href: window.location.href, title: document.title, bodyLen: document.body?.innerHTML?.length ?? 0 })'
```

Treat the context as stale when the sanity eval reports blank or `about:blank`, or `bodyLen` is `0`, while `CURRENT_URL` or the snapshot still shows a real page.

## Recovery

Recover once in the same live session. Do not loop.

```bash
export AGENT_BROWSER_SESSION="${AGENT_BROWSER_SESSION:?unset}"
CURRENT_URL="$(agent-browser get url 2>/dev/null || true)"
[ -n "$CURRENT_URL" ] && [ "$CURRENT_URL" != "about:blank" ] || CURRENT_URL="$TARGET_URL"
[ -f "${AGENT_BROWSER_STATE_FILE:-}" ] && agent-browser state load "$AGENT_BROWSER_STATE_FILE"
agent-browser open "$CURRENT_URL"
agent-browser wait --load networkidle || agent-browser wait 2000
agent-browser snapshot -i -C
agent-browser eval 'JSON.stringify({ href: window.location.href, title: document.title, bodyLen: document.body?.innerHTML?.length ?? 0 })'
```

If the retry still looks stale, mark the area blocked and keep the report honest. Do not assume a second reopen or daemon restart will fix it.

## Working style

- Keep shell blocks small and self-contained.
- Re-snapshot after every DOM change.
- Prefer `get url`, `get title`, `get count`, `get text`, and `wait --url` when they answer the question directly.
- Keep `eval` for structured extraction or checks the command surface cannot express cleanly.

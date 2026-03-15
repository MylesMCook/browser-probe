# Recovery

Use this file when the exploratory run gets derailed. Recover cleanly, then continue or mark the area blocked.

## Stale `eval` context

Symptoms:
- `snapshot -i -C` shows a real page, but `eval` reports blank fields, `about:blank`, or `bodyLen: 0`
- `get url` returns a real route, but `eval` behaves as though the page is empty

Recovery:

```bash
export AGENT_BROWSER_SESSION="${AGENT_BROWSER_SESSION:?unset}"
export AGENT_BROWSER_STATE_FILE="${AGENT_BROWSER_STATE_FILE:-/tmp/${AGENT_BROWSER_SESSION}-auth-state.json}"
CURRENT_URL="$(agent-browser get url 2>/dev/null || true)"
[ -n "$CURRENT_URL" ] && [ "$CURRENT_URL" != "about:blank" ] || CURRENT_URL="$TARGET_URL"
[ -f "$AGENT_BROWSER_STATE_FILE" ] && agent-browser state load "$AGENT_BROWSER_STATE_FILE"
agent-browser open "$CURRENT_URL"
agent-browser wait --load networkidle || agent-browser wait 2000
agent-browser snapshot -i -C
agent-browser eval 'JSON.stringify({ href: window.location.href, title: document.title, bodyLen: document.body?.innerHTML?.length ?? 0 })'
```

Retry once. If the context still looks stale, mark the area blocked. Do not keep reopening the page.

## Stale refs

Symptoms:
- click or fill fails on a ref that just existed
- the page changed after a modal, menu, or route transition

Recovery:

```bash
export AGENT_BROWSER_SESSION="${AGENT_BROWSER_SESSION:?unset}"
agent-browser snapshot -i -C
```

Do not reuse stale refs.

## Page still loading or unstable

Recovery:

```bash
export AGENT_BROWSER_SESSION="${AGENT_BROWSER_SESSION:?unset}"
agent-browser wait --load networkidle || agent-browser wait 2000
agent-browser snapshot -i -C
```

If the app never stabilizes, note it as blocked or flaky rather than pretending the state is settled.

## Opened a new tab or navigated somewhere unexpected

Recovery:

```bash
export AGENT_BROWSER_SESSION="${AGENT_BROWSER_SESSION:?unset}"
agent-browser tab
```

Switch to the correct tab, then re-snapshot.

## Modal or overlay trapped the run

Try, in order:

```bash
export AGENT_BROWSER_SESSION="${AGENT_BROWSER_SESSION:?unset}"
agent-browser press Escape
agent-browser snapshot -i -C
```

If that fails, use the overlay pattern file or click the obvious dismiss control after a fresh snapshot.

## Console-visible failure with no obvious UI repro

Recovery:

```bash
export AGENT_BROWSER_SESSION="${AGENT_BROWSER_SESSION:?unset}"
agent-browser console
agent-browser errors
agent-browser screenshot --annotate ./probe-recovery.png
```

Record it as a console-only issue or a blocked investigation, depending on what you can confirm.

## Unexpected auth wall

If credentials are missing:
- stop and mark the auth-gated area blocked
- continue only with public or already-accessible routes

If the run should be authenticated and you have credentials, switch to the appropriate auth flow before continuing.

## Browser process or local session got weird

First, close the active session cleanly:

```bash
export AGENT_BROWSER_SESSION="${AGENT_BROWSER_SESSION:?unset}"
agent-browser close
```

Then reopen the target in the same named session. If you have an explicit auth checkpoint, load it before reopening:

```bash
export AGENT_BROWSER_STATE_FILE="${AGENT_BROWSER_STATE_FILE:-/tmp/${AGENT_BROWSER_SESSION}-auth-state.json}"
[ -f "$AGENT_BROWSER_STATE_FILE" ] && agent-browser state load "$AGENT_BROWSER_STATE_FILE"
agent-browser open "$TARGET_URL"
agent-browser wait --load networkidle || agent-browser wait 2000
agent-browser snapshot -i -C
```

Do not assume an out-of-band daemon restart is the right fix.

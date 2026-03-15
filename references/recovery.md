# Recovery

Use this file when the exploratory run gets derailed. Recover cleanly, then continue or mark the area blocked.

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

Then reopen the target in the same named session and continue. Do not assume an out-of-band daemon restart is the right fix.

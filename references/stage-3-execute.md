# Stage 3 - Execute

Run the Stage 2 plan in priority order. Keep evidence proportional to the mode and the confidence level of the finding.

## Session and target discipline

Use the same `AGENT_BROWSER_SESSION` for the whole run, but do not treat it as a guarantee that `eval` still points at the last real page.

```bash
export AGENT_BROWSER_SESSION="${AGENT_BROWSER_SESSION:?unset}"
export AGENT_BROWSER_STATE_FILE="${AGENT_BROWSER_STATE_FILE:-/tmp/${AGENT_BROWSER_SESSION}-auth-state.json}"
agent-browser open "$TARGET_URL"
agent-browser wait --load networkidle
agent-browser snapshot -i -C
```

For non-local targets, prefer:

```bash
export AGENT_BROWSER_SESSION="${AGENT_BROWSER_SESSION:?unset}"
agent-browser --allowed-domains "$TARGET_HOST" open "$TARGET_URL"
```

If `TARGET_HOST` is empty, skip `--allowed-domains`, note the gap, and continue read-only.

Before any block that depends on `eval`, run the preflight in [session-preflight.md](session-preflight.md). If it detects a stale context, do one same-session reopen/retry and then mark the area blocked if the retry still fails.

## Locale and selector discipline

If the app is not English-first, rely less on text matching and more on:
- refs from `snapshot -i -C`
- roles
- labels
- structural selectors

Use text selectors only when the text is stable and visible.

## Execution loop

For each test-plan item:

1. Navigate to the relevant route or state.
2. Snapshot with `-i -C`.
3. Run the session preflight if the next step depends on `eval`.
4. Execute the test.
5. Re-snapshot after any DOM change.
6. Classify the result as pass, fail, flaky, blocked, or skipped.

Retry one time after a short wait before calling something a confirmed failure:

```bash
export AGENT_BROWSER_SESSION="${AGENT_BROWSER_SESSION:?unset}"
agent-browser wait 2000
```

If it passes on retry, call it flaky.

If the failure was a stale `eval` context rather than app behavior, use [recovery.md](recovery.md) instead of piling on more waits.

## Evidence rules by mode

### Smoke

- chat-first
- one screenshot only when it materially helps
- no trace or video unless the bug is otherwise unclear

### Balanced

- take an annotated screenshot for confirmed failures when layout or state matters
- use `console` and `errors` for investigation before escalating
- use `inspect`, `trace`, or `record` only if the issue cannot be explained cleanly without them

### Bug-hunt

- go deeper on edge cases and responsive behavior
- use extra screenshots when needed
- use `trace` or `record` for timing-sensitive failures
- still avoid artifact theater; collect evidence because it helps, not because it looks thorough

## Debug escalation order

Use this order:

1. `console`
2. `errors`
3. `screenshot --annotate`
4. `inspect`
5. `trace`
6. `record`

## Mobile and responsive checks

Do not use CSS hacks. Use first-class emulation:

```bash
export AGENT_BROWSER_SESSION="${AGENT_BROWSER_SESSION:?unset}"
agent-browser set viewport 390 844 2
agent-browser reload
agent-browser wait --load networkidle
agent-browser snapshot -i -C
```

Or use a named device when device semantics matter:

```bash
export AGENT_BROWSER_SESSION="${AGENT_BROWSER_SESSION:?unset}"
agent-browser set device "iPhone 15 Pro"
agent-browser reload
agent-browser wait --load networkidle
agent-browser snapshot -i -C
```

Responsive checks are:
- optional in `smoke`
- situational in `balanced`
- expected in `bug-hunt` when the app claims responsive support or the user cares about mobile

## Pattern files

Load only the patterns that fit the discovered UI:

| Found in Stage 1 | Load |
| --- | --- |
| navigation | `references/patterns/navigation.md` |
| auth | `references/patterns/auth.md` |
| forms | `references/patterns/forms.md` |
| tables/cards/lists | `references/patterns/data-display.md` |
| overlays | `references/patterns/overlays.md` |
| search/filter | `references/patterns/search-filter.md` |

If a pattern file is missing or stale, skip it and note the gap instead of blocking the whole probe.

If the run derails, use `references/recovery.md`.

# browser-probe

Exploratory browser-testing skill built on `agent-browser`.

Use it for:

- smoke tests
- scoped feature verification
- balanced exploratory QA
- deeper single-agent bug hunting

Do not use it for:

- repro-heavy issue packages
- step-by-step evidence for every issue
- formal handoff reports for another team

## Requirements

- `agent-browser 0.20.x`

## Session contract

- Reuse one `AGENT_BROWSER_SESSION` for the run.
- Treat that session as live isolation, not as a guarantee that `eval` keeps the same page context across later shell blocks.
- Before any `eval`-heavy block, run the preflight in [references/session-preflight.md](references/session-preflight.md).
- After successful auth, save explicit state to `AGENT_BROWSER_STATE_FILE` if later recovery needs to reopen the page.

## What it does

`browser-probe` runs in four stages:

1. model the app or scoped feature
2. generate failure hypotheses and strength checks
3. execute targeted tests
4. report findings, flakes, blocked areas, and verified strengths

## Modes

- `smoke`: quick core-path verification
- `balanced`: default exploratory pass
- `bug-hunt`: deeper adversarial testing

## Install

```bash
npx skills add MylesMCook/browser-probe
```

## Example

```text
Use browser-probe in balanced mode on the checkout flow. Verify the happy path, look for obvious edge-case failures, and note what works well too.
```

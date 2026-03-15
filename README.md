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
npx skills add https://github.com/MylesMCook/browser-probe --skill browser-probe
```

## Example

```text
Use browser-probe in balanced mode on the checkout flow. Verify the happy path, look for obvious edge-case failures, and note what works well too.
```

# browser-probe

I made this because I wanted something between a smoke test and a full bug-report machine.

I wanted a skill that would:

- understand the app or feature
- form a few good hypotheses about what might break
- test them
- tell me what is broken
- also tell me what actually works well

## Why I use it

A lot of browser testing falls into two extremes:

- too shallow to be useful
- too heavy to run casually

`browser-probe` is the middle layer.

It is for exploratory QA when you want a real read on quality without turning the run into formal issue packaging.

## Boundaries

- `agent-browser` is the low-level control layer
- `browser-probe` is the exploratory testing layer
- a repro-first handoff workflow is the formal bug-packaging layer

## What it does

The skill runs in four stages:

1. model the app or scoped feature
2. generate failure hypotheses and strength checks
3. run targeted tests
4. report findings, flakes, blocked areas, and verified strengths

## The three modes

### Smoke

For quick checks and core-path verification.

### Balanced

The default mode.

This is the one I wanted most: a product-minded pass that looks for likely failures but also notices when important paths are working cleanly.

### Bug-hunt

For deeper adversarial testing when you want to push harder on edge cases and responsive behavior.

## What I want back

I do not just want a list of failures. I want to know:

- what broke
- what felt flaky
- what I could not verify
- what worked better than expected

That gives me a better sense of the product than a flat bug list.

## Under the hood

`browser-probe` uses `agent-browser` for execution, but the useful part is the judgment layer:

- where to look first
- what to test next
- how hard to push
- when a failure is real
- when a working flow is worth calling out

Like most of the tools I make, this started as a thing I wanted for myself. It is public in case it helps someone else.

## Install

This repo root is the skill directory.

```bash
npx skills add https://github.com/MylesMCook/browser-probe --skill browser-probe
```

It expects `agent-browser 0.20.x`.

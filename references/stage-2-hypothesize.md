# Stage 2 - Generate hypotheses

Use the Stage 1 model to produce a small, useful test plan. This stage now includes both failure hypotheses and strength checks.

## Goal

Generate:
- prioritized failure hypotheses
- a small set of things that should work well if the app is healthy

`browser-probe` is not just hunting for breakage. It should also verify strengths.

## Build hypotheses from these sources

### A. Core or scoped flows

Look at the critical flows from Stage 1 and ask:
- where does state hand off between steps?
- where could navigation or validation lose context?
- where could async loading leave the UI stuck, stale, or misleading?

### B. Component-specific weak points

Only use the component families actually found:

- forms: validation clears, submit behavior, required fields, field persistence
- tables and lists: sorting, filtering, pagination, row selection, empty states
- navigation: active states, deep links, back-button behavior
- overlays: focus trap, dismissal, scroll locking, nested overlay behavior
- auth: redirects, session persistence, blocked protected routes

### C. Integration surfaces

Focus on what is observable without inventing devtools magic:
- loading states
- empty states
- retries
- stale or partial UI after an action

If a hypothesis would require network interception you do not have, mark it as not directly testable and do not pretend otherwise.

### D. Responsive or adversarial behavior

Only include these when the mode or task warrants it:
- `balanced`: a few obvious edge cases
- `bug-hunt`: responsive checks plus adversarial inputs that can surface visual or UX breakage

## Add strength checks

Every run should include a few "this should work cleanly" targets:
- the primary happy path
- one navigation path
- one key display or search/filter path

Strength checks stop the report from becoming a pile of negatives with no calibration.

## Hypothesis counts by mode

- `smoke`: 3 to 5 total items, mostly happy-path verification with a couple of obvious failure risks
- `balanced`: 6 to 10 total items, mixed failure hypotheses and strength checks
- `bug-hunt`: 10 to 15+ total items, including edge cases and responsive behavior

## Write the test plan

Use this format:

```text
TEST PLAN
Mode: [smoke | balanced | bug-hunt]

Failure hypotheses:
1. [priority] [feature] - [specific predicted failure]
   Test: [what to do]

Strength checks:
1. [feature] - [what should work cleanly]
   Test: [what to do]
```

Keep the tests concrete. Stage 3 should be able to execute them directly.

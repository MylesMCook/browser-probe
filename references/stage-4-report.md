# Stage 4 - Report

Transform Stage 3 into a concise exploratory-testing report. The report must reflect both quality risks and quality signal.

## Always include

- mode used
- scope covered
- confirmed findings by severity
- flaky findings
- blocked or skipped areas
- verified strengths

## Severity definitions

- `Critical`: the core purpose is blocked
- `High`: a major feature is broken or badly degraded
- `Medium`: the task still works, but the UX or reliability is notably poor
- `Low`: minor but real defect
- `Flaky`: failed once, then passed on retry

## Report format

Use this shape:

```text
BROWSER PROBE REPORT
Mode: [smoke | balanced | bug-hunt]
Scope: [full app | feature]
Target: [URL or route]

Confirmed findings:
- [Severity] [feature] - [what failed]
  Evidence: [screenshot / console / trace / none]
  Notes: [brief repro or root-cause hypothesis]

Flaky findings:
- [feature] - [what failed once and passed on retry]

Blocked or skipped:
- [feature or area] - [why it was blocked or skipped]

Verified strengths:
- [feature or flow] - [what worked cleanly]

Overall read:
- [one short paragraph that answers: should the user trust this area?]
```

## Writing rules

- Lead with the most important failures.
- Do not pad the report to look busier than the run was.
- If nothing broke, say so clearly and still list the verified strengths.
- If the user scoped the run, name the out-of-scope areas under blocked or skipped rather than implying they were tested.
- Keep the evidence note honest. If you did not capture a screenshot, say `none`.

## When the run points to a formal handoff workflow instead

If the run reveals enough issues that the user clearly now needs handoff-grade repro packages, say so explicitly at the end:

```text
Follow-up recommendation:
- This area now warrants a more formal repro-first QA pass if the goal is issue handoff with repro artifacts.
```

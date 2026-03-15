---
name: browser-probe
description: Exploratory browser testing for real web apps. Use when the user wants a smoke test, scoped verification, exploratory QA, or a balanced read on what works, what feels off, and what is blocked. Supports smoke, balanced, and bug-hunt modes. Uses agent-browser as the execution substrate. Do not use when the user explicitly wants repro-heavy bug artifacts or handoff-grade issue packages; use a more formal repro-first handoff workflow for that.
allowed-tools: Bash
---

# browser-probe

`browser-probe` is the exploratory testing skill.

This skill assumes the current `agent-browser 0.20.x` command surface.

Its job is to:
- model the app or scoped feature
- form hypotheses about likely breakage
- test those hypotheses
- report confirmed findings, flakes, blocked areas, and verified strengths

It is intentionally lighter than a repro-first handoff workflow.

## When to use this skill

Use `browser-probe` when the user wants:
- a smoke test
- feature verification
- exploratory QA
- a sanity check
- a high-signal read on what works and what feels off

Do not use `browser-probe` when the user wants:
- step-by-step repro packages
- screenshots for every step of every issue
- repro videos by default
- handoff-grade bug reports for another team

Those requests belong to a more formal repro-first handoff workflow.

If your environment has a dedicated skill for that, route there. Otherwise say this request is out of scope for `browser-probe`.

## Mode selection

Choose one mode at the start and state it in the final report:

| Mode | When to use it | What changes |
| --- | --- | --- |
| `smoke` | User says "smoke", "quick", "just verify", or the feature is tiny | Test core flows only. Minimal evidence. |
| `balanced` | Default | Explore likely failures and likely strengths. Evidence stays proportional. |
| `bug-hunt` | User says "bug hunt", "try to break it", "go hard", or wants deeper adversarial testing | Push harder on edge cases, responsive behavior, and failure investigation. Still single-agent in this pass. |

If the user scopes a feature, apply the chosen mode only to that scope.

## Session isolation

Every run gets one named `agent-browser` session. Reuse it across all shell calls in that run. Follow the `agent-browser` substrate rules for ref freshness and session hygiene.

At the start of the run:

```bash
RAND_SUFFIX="$(openssl rand -hex 2 2>/dev/null || printf '%04x' $((RANDOM % 65536)))"
export AGENT_BROWSER_SESSION="probe-$(date +%s)-$RAND_SUFFIX"
```

For every shell block that calls `agent-browser`, export the same session first:

```bash
export AGENT_BROWSER_SESSION="${AGENT_BROWSER_SESSION:?unset}"
agent-browser open "$TARGET_URL"
agent-browser wait --load networkidle
agent-browser snapshot -i -C
```

Do not let separate shell calls drift onto different sessions.

## Safe probing posture

- Use direct `agent-browser`, not `npx agent-browser`.
- Use `snapshot -i -C` everywhere.
- Re-snapshot after any DOM change.
- For non-local targets, default to `--allowed-domains`.
- Keep the run read-mostly unless the user explicitly asked for state-changing behavior or the scoped test requires it.
- If the environment already has an action policy or confirmation categories, use `--action-policy` and `--confirm-actions` instead of freehand mutating a real site.

## Resolve the target

If the user provided a URL, use it.

If not, probe the common local dev ports:

```bash
for port in 3000 3001 4000 4200 5173 5174 8080 8000; do
  if curl -s -o /dev/null --connect-timeout 1 "http://localhost:$port"; then
    export TARGET_URL="http://localhost:$port"
    break
  elif curl -sk -o /dev/null --connect-timeout 1 "https://localhost:$port"; then
    export TARGET_URL="https://localhost:$port"
    break
  fi
done
```

If nothing responds, stop and report that the app is not reachable.

After `TARGET_URL` is known, derive `TARGET_HOST` once for later safety flags:

```bash
export TARGET_HOST="$(python3 - <<'PY'
from urllib.parse import urlparse
import os
print(urlparse(os.environ['TARGET_URL']).hostname or '')
PY
)"
```

## The four stages

Do not skip the stages. Keep them concise, but preserve the flow:

1. Stage 1: model the app or scoped feature
2. Stage 2: generate failure hypotheses and strength checks
3. Stage 3: execute targeted tests with proportional evidence
4. Stage 4: report findings, verified strengths, flakes, and blocked areas

Read and follow these files in order:

| Stage | File |
| --- | --- |
| 1 | [references/stage-1-model.md](references/stage-1-model.md) |
| 2 | [references/stage-2-hypothesize.md](references/stage-2-hypothesize.md) |
| 3 | [references/stage-3-execute.md](references/stage-3-execute.md) |
| 4 | [references/stage-4-report.md](references/stage-4-report.md) |

If the run goes sideways mid-test, use [references/recovery.md](references/recovery.md).

## Evidence rules

- `smoke`: chat-first, minimal screenshots
- `balanced`: enough evidence to support the conclusion, no artifact theater
- `bug-hunt`: more screenshots and targeted debug capture; traces or video only when they genuinely help

This skill is not obligated to generate an artifact directory. If the user wants a formal evidence pack, route to a repro-first handoff workflow instead.

## Pattern files

Load pattern files only for component types that Stage 1 actually found:

| Area | Pattern |
| --- | --- |
| Navigation | [references/patterns/navigation.md](references/patterns/navigation.md) |
| Auth | [references/patterns/auth.md](references/patterns/auth.md) |
| Forms | [references/patterns/forms.md](references/patterns/forms.md) |
| Data display | [references/patterns/data-display.md](references/patterns/data-display.md) |
| Overlays | [references/patterns/overlays.md](references/patterns/overlays.md) |
| Search and filters | [references/patterns/search-filter.md](references/patterns/search-filter.md) |

This pass is single-agent only. Do not invent swarm behavior.

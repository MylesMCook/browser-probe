# browser-probe evals

These evals are meant to keep `browser-probe` differentiated from both `agent-browser` and `dogfood`.

## What to check

For prompts where the skill should trigger:

1. The response chooses exploratory testing rather than raw browser automation.
2. It defaults to balanced mode unless the prompt clearly asks for smoke or bug-hunt.
3. It includes both likely failure areas and things that should work well.
4. It keeps evidence proportional to the request.
5. It treats `AGENT_BROWSER_SESSION` as live isolation only and uses the session preflight before `eval`-heavy work.
6. It prefers `get url`, `get title`, `get count`, `get text`, and `wait --url` when those are enough.

For prompts where the skill should not dominate:

1. It yields to `agent-browser` for narrow automation tasks.
2. It yields to `dogfood` for repro-heavy handoff requests.
3. It does not fall back to stale daemon or CSS-emulation guidance.
4. It does not assume `AGENT_BROWSER_SESSION_NAME` is the default persistence layer.

## Suggested scoring dimensions

- Trigger accuracy
- Mode selection
- Exploratory depth
- Dogfood boundary discipline
- Evidence proportionality
- Strength reporting
- Session recovery discipline
- Non-local safety guardrails

## How to use the prompt set

Keep the CSV small and practical. Add prompts when the skill starts drifting into generic automation or artifact-heavy QA.

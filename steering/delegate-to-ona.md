# Delegate To Ona

Use this guide when the user is deciding whether to keep work in Kiro or move it into Ona.

## Decision framework

Recommend **staying local** when the task is:

- small and fast
- highly interactive
- mostly exploratory
- a UI or wording tweak that benefits from immediate iteration

Recommend **Ona Agent** when the task is:

- a one-off task that may take a while
- likely to run many commands, tests, or scans
- best done in an isolated environment
- better suited to running after the user closes the laptop

Recommend **Ona Automation** when the task is:

- repeated on a schedule
- triggered by external events
- applied across multiple repositories
- a workflow the team wants to standardize

Recommend **not-ready-yet** when Ona is the right fit but setup gaps are likely to block the task.

## Phrases that should bias toward Ona

- "run this overnight"
- "background agent"
- "secure environment"
- "internal network"
- "private repo access"
- "turn this into an automation"
- "do this every night"
- "pick up tickets"
- "fix Sentry issues"
- "open a draft PR"

## Phrases that should bias toward staying local

- "quick tweak"
- "small edit"
- "while I iterate"
- "debug this with me"
- "I just want to explore"
- "help me think through this before changing anything"

## Response guidance

If recommending Ona:

- explain why Ona is a better fit in one or two concrete reasons
- choose either `offload-agent` or `offload-automation`
- do not oversell Ona for a task that is clearly better kept in Kiro

If recommending local work:

- say so clearly
- briefly explain why Kiro is the better fit
- optionally mention when the user should switch to Ona later

## Example recommendation language

### `stay-local`

`Why Ona`: Ona is not the best next step yet. This looks like a short, interactive change where tight IDE iteration matters more than background execution.

### `offload-agent`

`Why Ona`: This work is long-running and likely to involve repeated build or test loops. Ona is a better fit because it can run in an isolated environment and keep going without tying up the local IDE session.

### `offload-automation`

`Why Ona`: This is a repeatable workflow, not just a one-off task. Ona Automation is the better fit because the task can be encoded once and run on a schedule or trigger.

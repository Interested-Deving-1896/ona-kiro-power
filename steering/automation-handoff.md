# Automation Handoff

Use this guide when the user's request should become an Ona Automation rather than a one-off Ona Agent task.

This guide is for recommending or launching **existing** automation behavior. It is not a license to invent brand new automation task definitions through CLI commands in this version.

## When to choose automation

Prefer `offload-automation` when the task is:

- repeated on a schedule
- repeated across repositories
- triggered by pull requests, webhooks, or issue events
- a maintenance workflow the team wants to standardize

Examples:

- nightly dependency cleanup
- daily backlog pickup
- recurring Sentry triage
- scheduled test or validation sweeps
- repeated docs or migration work across repositories

## What to hand off

When recommending automation, produce a brief that is ready to paste into Ona Automation setup.

The `Ona prompt` section should include:

- the task to perform
- what should trigger it
- what success looks like
- whether it should open a draft PR
- what external context it may need

## Default automation framing

Unless the user specifies otherwise, assume:

- draft PRs are safer than ready-to-merge PRs
- small, reviewable changes are preferred
- repeated work should be explicit about success criteria

## Existing task start flow

If the user explicitly wants to run an existing automation task:

- require an environment ID or create/select the environment first
- offer to list tasks with `ona automations task list -e <environment-id> -o json`
- ask the user to confirm before listing if Kiro requires command approval
- after identifying the task, offer `ona automations task start <task-ref> -e <environment-id> --dont-wait`
- require explicit confirmation before starting the task

This flow is for **existing** tasks only.

## Example structure for automation handoff

### `Recommended path`

`Recommended path`: Offload this to an Ona Automation. This is repeatable work that will be easier to maintain as a scheduled or triggered workflow than as a one-off prompt.

### `Ona prompt`

Use a compact brief like this:

```text
Run this workflow as an Ona Automation.

Goal:
- <what the automation should accomplish>

Trigger:
- <schedule, PR event, webhook, or manual trigger>

Execution rules:
- Work in the relevant repository or project
- Keep changes small and reviewable
- Run the smallest relevant checks
- Open a draft PR with a concise summary

Success criteria:
- <what should be true when the automation finishes>
```

## When not to choose automation

Do not choose automation when the user only needs:

- a one-off investigation
- a single implementation pass
- work that still needs interactive shaping before repetition makes sense

Also do not claim that this power can create brand new automation tasks or services directly through the CLI in this version.

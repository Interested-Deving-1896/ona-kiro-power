---
name: "ona"
displayName: "Offload to Ona"
description: "Detect when work belongs in Ona, resolve the current repo to a project, and use the Ona CLI to launch environments, start one-off AI executions, or run saved automations with explicit confirmation."
keywords: ["ona", "ona cli", "run in ona", "launch in ona", "create ona environment", "ona project", "ona automation", "prepare repo for ona", "dev container for ona", "gitpod", "linear in ona", "sentry in ona", "draft pr in ona", "private network in ona"]
---

# Onboarding

Use this power when the user wants help deciding whether work belongs in Ona, launching the next step with the Ona CLI, or preparing a repository so it is ready for Ona later.

When this power first activates:

1. Decide whether the task should stay local or move to Ona.
2. If the user wants direct Ona execution and Kiro can run shell commands, use the safe preflight checks listed below.
3. If the Ona CLI is unavailable but the user wants to prepare the repository for Ona, switch to the local setup path instead of stopping.

## Safe preflight checks

When command execution is available, use these in order:

- `command -v ona`
- `ona whoami -o json`
- `git remote get-url origin`
- `ona project list --limit 1000 -o json`
- `ona environment list -a -o json`

Do not run side-effecting commands during preflight.

For project selection specifically:

- do not reason directly from the raw `ona project list --limit 1000 -o json` output in chat
- after preflight, use the combined filtering and ranking command from `steering/login-and-readiness.md`
- only present repo-matching, ranked candidates to the user

## Side-effecting commands

Only run these after explicit user confirmation:

- `ona login`
- `ona login --no-browser`
- `ona project create <repo-url> ...`
- `ona environment create <project-id> --dont-wait --set-as-context ...`
- `ona environment start <environment-id> --set-as-context`
- `ona ai automation execute - --environment-id <environment-id>`
- `ona ai automation start <automation-id> --project <project-id>`
- `ona automations task list -e <environment-id> -o json`
- `ona automations task start <task-ref> -e <environment-id> --dont-wait`

When building a command:

- use only flags that were validated from the live CLI help or from a successful earlier run
- do not assume every command supports `-o json`
- for `ona environment create`, prefer the plain command because it already prints the environment ID directly

## Response contract

For Ona-oriented responses, keep this section order:

1. `Why Ona`
2. `Recommended path`
3. `Readiness`
4. `Next step`
5. `Ona prompt`
6. `Additional setup needed`

`Next step` should say whether the power can run the action now or what setup or confirmation is still needed.

## What not to claim

This version is a CLI-backed launcher and local setup helper. Do not claim that it can:

- run arbitrary `ona environment exec` commands
- create brand new automation task definitions or services from natural language alone
- write automation config directly into an existing remote environment
- open an editor automatically unless the user asks

# Steering Instructions

Use steering files for the detailed workflows and best practices instead of repeating them here:

- Deciding whether to stay local or move to Ona: `steering/delegate-to-ona.md`
- Determining readiness, project resolution, Git auth, or integration gaps: `steering/login-and-readiness.md`
- Running the CLI-backed launch flow: `steering/execution-flow.md`
- Preparing a repository locally for Ona: `steering/local-setup-flow.md`
- Turning repeated work into an automation recommendation: `steering/automation-handoff.md`
- Checking whether a repo is ready for project creation: `steering/repo-readiness.md`

# Global Rules

- Prefer Ona for long-running, secure, internal-network-dependent, or repeatable work.
- Prefer staying local for short, interactive work.
- Use a **project-first** strategy whenever a project match exists.
- Filter and rank matching projects before presenting them. Do not dump raw `ona project list` output into the chat.
- For multiple project matches, use the explicit combined ranking command from `steering/login-and-readiness.md` instead of improvising from the raw JSON.
- Treat one-off long-running requests as prompt-driven AI execution, not as recurring automation.
- Scope environment reuse to the current Kiro session by default. Do not attach a new task to some other existing environment unless the user explicitly asks.
- Ask for confirmation before starting a side-effecting flow, not before every single command inside an already approved flow.
- Treat one-off AI executions as background handoffs. Once Ona has accepted the run, report that it was handed off instead of waiting for completion in chat.
- Separate Ona login from Git auth and integrations in both reasoning and user-facing output.
- If the user is trying to get a repo Ona-ready, do not block on the CLI; offer local Dev Container and automation setup instead.
- Use the canonical setup prompt for local repo-preparation flows rather than inventing a shorter one.
- Preserve the user's original prompt for one-off Ona execution instead of replacing it with a repo task name.
- Keep one-off AI execution YAML ephemeral. Only suggest saving YAML under `.ona/` when the user wants a reusable workflow.

---
name: "ona"
displayName: "Offload to Ona"
description: "Detect when work belongs in Ona, resolve the current repo to a project, and use the Ona CLI to launch environments or existing automation tasks with explicit confirmation."
keywords: ["ona", "gitpod", "background agent", "run overnight", "automation", "secure environment", "draft pr", "pull request", "linear", "sentry", "notion", "internal access", "long-running", "offload", "agent", "repeatable work", "recurring task", "vpc", "private network", "launch environment", "use ona cli"]
---

# Onboarding

## What this power does

Use this power when the user wants to decide whether work should move from Kiro into Ona, and when possible, launch the next step through the Ona CLI.

This version is a **CLI-backed launcher**:

- it can detect local Ona readiness automatically
- it prefers existing Ona projects over raw repository URLs
- it ranks matching projects instead of hard-truncating them
- it can offer to run `ona login`
- it can offer to create an Ona project
- it can offer to create an Ona environment
- it can offer to start an existing automation task

This version is **not** a general remote execution engine. Do not claim that it can:

- run arbitrary `ona environment exec` commands
- write `automations.yaml` into an environment
- create brand new automation tasks or services from natural language alone
- open an editor automatically unless the user asks

## Step 1: Decide whether Ona is the right fit

Prefer **staying local** when the task is:

- small and fast
- highly interactive
- exploratory debugging
- a quick UI or wording tweak

Prefer **Ona** when the task is:

- long-running
- secure or isolated
- dependent on internal or private network access
- likely to involve repeated build, test, or scan loops
- repeated on a schedule or across repositories

## Step 2: Use the CLI-backed execution states

Classify the request into one primary state:

- `stay-local`
- `needs_cli`
- `needs_ona_login`
- `needs_project_resolution`
- `needs_git_auth`
- `needs_integration`
- `ready_to_create_environment`
- `ready_to_start_existing_task`

Use these states to drive both the explanation and the next action.

## Step 3: Run safe preflight checks automatically when command execution is available

Safe preflight commands:

1. `command -v ona`
2. `ona whoami -o json`
3. `git remote get-url origin`
4. `ona project list -o json`
5. `ona environment list -a -o json`

Use them in this order.

Interpretation rules:

- if the task is not a good Ona fit -> `stay-local`
- if `command -v ona` fails -> `needs_cli`
- if `ona whoami -o json` fails -> `needs_ona_login`
- if the current repo resolves to exactly one Ona project -> use that project
- if it resolves to multiple projects -> `needs_project_resolution`
- if it resolves to no project -> inspect repo readiness before offering project creation

Do not run side-effecting commands during preflight.

## Step 4: Resolve projects in a project-first way

When a current Git repository is available:

1. Read `git remote get-url origin`
2. Normalize GitHub SSH and HTTPS URLs before matching
3. Inspect `ona project list -o json`
4. Inspect `ona environment list -a -o json` for project recency
5. Match on project repository metadata, not just project name

Use the repository URL under the project's initializer metadata when available.

Matching rules:

- exactly one match -> use that project ID
- 2 to 10 matches -> show all matches
- more than 10 matches -> show a ranked top list first and offer to show all
- zero matches -> check `.devcontainer/devcontainer.json`

If no project exists:

- if `.devcontainer/devcontainer.json` exists, offer to create a project after confirmation
- if it does not exist, stop and explain that the repo is not ready for automatic project creation

Project ranking rules:

- rank exact repo matches above everything else
- then rank by recent personal usage when that can be derived from environments
- then rank by user-supplied project-name hints if present
- then fall back to name order

For large match sets:

- show the top 5 first
- tell the user how many more matches exist
- support follow-ups like `show all`, `filter <text>`, and `use <project-id>`

Do not silently choose between multiple projects.

## Step 5: Require confirmation before any side effect

All side-effecting CLI actions require explicit user confirmation.

Supported confirmed commands:

- `ona login`
- `ona login --no-browser`
- `ona project create <repo-url> ...`
- `ona environment create <project-id> --dont-wait --set-as-context ...`
- `ona automations task list -e <environment-id> -o json`
- `ona automations task start <task-ref> -e <environment-id> --dont-wait`

When a mutating action is available:

- name the exact command you intend to run
- explain why that command is the next step
- ask for confirmation before running it

## Step 6: Keep Ona login, Git auth, and integrations separate

Treat these as different readiness layers:

1. **Ona login**
   - Browser path: `https://app.ona.com`
   - CLI path: `ona login`
   - No-browser option: `ona login --no-browser`
   - PAT fallback: `ona login --token <token>`

2. **Git provider authentication**
   - Needed for cloning, pushing, and PR workflows
   - Do not imply that Ona login covers Git auth
   - Point users to `https://app.ona.com/settings/git-authentications` when relevant

3. **Optional integrations**
   - Linear, Sentry, and Notion are only required when the requested workflow depends on them

## Step 7: Keep the structured response contract

For every Ona-oriented response, use these exact sections in order:

1. `Why Ona`
2. `Recommended path`
3. `Readiness`
4. `Next step`
5. `Ona prompt`
6. `Additional setup needed`

Rules:

- `Next step` should say whether the power can run the action now, or what setup/confirmation is still needed
- `Ona prompt` remains required even when CLI execution is available
- `Additional setup needed` should only mention blockers relevant to the requested task

## Step 8: Respect Kiro shell approvals

This power runs shell commands through Kiro's normal command approval system.

Guidance for the user:

- safe preflight checks may still need Kiro approval depending on their trust settings
- side-effecting commands should always be explicitly confirmed by the user before you run them
- do not ask the user to broadly trust wildcard commands unless they specifically want to optimize the workflow
- if many matching projects exist, the power should narrow the list interactively instead of forcing a guess

# When To Load Steering Files

- Deciding whether to stay local or move to Ona -> `steering/delegate-to-ona.md`
- Determining login state, project resolution, Git auth, or integration gaps -> `steering/login-and-readiness.md`
- Preparing or launching the CLI-backed flow -> `steering/execution-flow.md`
- Turning repeated work into an automation recommendation -> `steering/automation-handoff.md`
- Checking whether a repo is ready for project creation -> `steering/repo-readiness.md`

# Core Rules

- Prefer Ona for long-running, secure, internal-network-dependent, or repeatable work.
- Prefer staying local for short, interactive work.
- Use a **project-first** strategy whenever a project match exists.
- Ask for confirmation before running any side-effecting Ona CLI command.
- Separate Ona login from Git auth and integrations in both reasoning and user-facing output.
- Stop after successful environment creation unless the user explicitly wants the next step, such as starting an existing automation task.

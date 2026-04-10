# Offload to Ona

`Offload to Ona` is a custom Kiro power that helps users decide when work should move from the IDE into Ona, checks local readiness, resolves the current repository to an Ona project, and launches the next step with the Ona CLI after explicit confirmation.

This version is intentionally scoped:

- automatic local detection when command execution is available
- project-first matching against existing Ona projects
- confirmed CLI-backed login, project creation, environment creation, and existing-task start flows
- no MCP server requirement
- optional local checks for `command -v ona`, `ona whoami -o json`, `git remote get-url origin`, and `ona project list -o json`
- browser-first fallback when CLI checks are unavailable

## What it does

The power is designed for requests like:

- "Run this overnight in Ona"
- "This needs private network access. Should we move it to Ona?"
- "Turn this cleanup into an automation"
- "Pick up this Linear ticket in Ona"
- "Use Ona to fix this and raise a draft PR"

For matching requests, it classifies the task into one of these execution states:

- `stay-local`
- `needs_cli`
- `needs_ona_login`
- `needs_project_resolution`
- `needs_git_auth`
- `needs_integration`
- `ready_to_create_environment`
- `ready_to_start_existing_task`

For every offload recommendation, it returns:

- `Why Ona`
- `Recommended path`
- `Readiness`
- `Next step`
- `Ona prompt`
- `Additional setup needed`

## What the power may execute

The power separates commands into two classes.

### Safe preflight commands

These are used for detection and project resolution when Kiro allows command execution:

- `command -v ona`
- `ona whoami -o json`
- `git remote get-url origin`
- `ona project list -o json`

### Confirmed side-effect commands

These require explicit user confirmation before the power runs them:

- `ona login`
- `ona login --no-browser`
- `ona project create <repo-url> ...`
- `ona environment create <project-id> --dont-wait --set-as-context ...`
- `ona automations task list -e <environment-id> -o json`
- `ona automations task start <task-ref> -e <environment-id> --dont-wait`

This version does **not** run arbitrary `ona environment exec` commands or write automation config into environments.

## Installation

### Install from local path

1. Open Kiro.
2. Open the Powers panel.
3. Choose **Add power from Local Path**.
4. Select the repository root containing `POWER.md`.
5. Confirm installation.
6. Use **Try power** or ask a matching question in a new conversation.

### Install from GitHub

In Kiro:

1. Open the Powers panel.
2. Choose **Add power from GitHub**.
3. Enter this repository URL:

```text
https://github.com/gitpod-io/ona-kiro-power
```

4. Confirm installation.

The repository root contains `POWER.md`, so it is ready for GitHub-based install.

## How it shows up in Kiro

- After local-path install, it appears in the Powers panel immediately.
- After GitHub install, it appears as an installed custom power in the Powers panel.
- Auto-activation depends on the keywords in `POWER.md`.
- The onboarding flow can be exercised with **Try power** after installation.

## Kiro shell approvals

This power uses Kiro's normal shell-command approval flow.

Expected behavior:

- Kiro may ask the user to approve preflight checks
- mutating Ona CLI commands should only be run after the user explicitly confirms the action
- users do not need to grant broad wildcard trust for this version to work

Recommended approach:

- approve preflight commands as prompted
- approve side-effect commands intentionally when the power explains the exact command and why it needs to run it

## Login and setup expectations

This power treats readiness as three separate concerns.

### 1. Ona login

Preferred paths:

- browser: [app.ona.com](https://app.ona.com)
- CLI: `ona login`
- no-browser CLI: `ona login --no-browser`
- PAT fallback: `ona login --token <token>`

PATs can be created at:

- [app.ona.com/settings/personal-access-tokens](https://app.ona.com/settings/personal-access-tokens)

### 2. Git provider authentication

This is separate from Ona login. It is required for:

- cloning repositories
- pushing commits
- opening pull requests from Ona work

Relevant page:

- [app.ona.com/settings/git-authentications](https://app.ona.com/settings/git-authentications)

### 3. Optional integrations

Only needed when the task depends on them:

- Linear
- Sentry
- Notion

Users can manage integrations in Ona settings:

- [app.ona.com/settings/org-integrations](https://app.ona.com/settings/org-integrations)

## Project-first behavior

When a Git repository is available, the power should prefer an existing Ona project over a raw repository URL.

The intended flow is:

1. detect the current `origin` remote
2. normalize the remote URL
3. compare it against `ona project list -o json`
4. use the matching project when there is exactly one match
5. ask the user to choose if there are multiple matches
6. offer project creation only when there is no match and the repo appears Ona-ready

The default minimum signal for automatic project creation is:

- `.devcontainer/devcontainer.json` exists

## Example prompts

These are good prompts to verify activation and response quality:

1. `Run this test suite overnight in Ona and open a draft PR if it passes.`
2. `This bug needs access to our internal staging network. Should we offload it to Ona?`
3. `Turn this dependency cleanup into a nightly Ona automation.`
4. `Use Ona to pick up my Linear ticket and prepare the implementation plan.`
5. `Should I keep this frontend tweak in Kiro or move it to Ona?`
6. `I want Ona to investigate this Sentry issue and propose a fix.`
7. `If this repo already has an Ona project, create an environment for it.`
8. `Use the Ona CLI to create the environment, but ask me before you run anything.`

## Suggested release flow

Use a GitHub-first rollout:

1. Validate locally with **Add power from Local Path**
2. Ask a few users to install from GitHub and test onboarding
3. Use GitHub tags or GitHub Releases for versioned updates
4. Consider a curated `kiro.dev/powers` listing later if adoption is strong

## Troubleshooting

### `ona` CLI is missing

The power should fall back cleanly:

- open [app.ona.com](https://app.ona.com)
- sign in there
- install the CLI later if command-line flows are needed

### `ona whoami` fails

The power should treat this as `needs_ona_login` and suggest:

```bash
ona login
```

Or, when a PAT is preferred:

```bash
ona login --token <token>
```

If browser login is inconvenient, the power can also offer:

```bash
ona login --no-browser
```

### The user is logged into Ona but still cannot push or open PRs

The likely gap is Git provider authentication, not Ona login. Direct the user to:

- [app.ona.com/settings/git-authentications](https://app.ona.com/settings/git-authentications)

### The task mentions Linear, Sentry, or Notion but the workflow is blocked

The likely gap is an optional integration. The power should call that out separately instead of presenting it as an Ona login problem.

### The repo does not match any Ona project

The power should not silently fall back to a raw repo workflow. It should:

- check whether `.devcontainer/devcontainer.json` exists
- offer `ona project create <repo-url> ...` only after confirmation
- otherwise explain that the repo is not ready for automatic project creation

### Multiple Ona projects match the same repository

The power should not guess. It should show concise candidates and ask the user to choose before creating an environment.

## Manual test cases

Validate these flows in Kiro:

1. authenticated user with a matching project
2. unauthenticated user who confirms `ona login`
3. no CLI installed
4. repo with multiple matching projects
5. repo with no matching project but a valid devcontainer
6. repo with no matching project and no devcontainer
7. PR-oriented request with missing Git auth
8. Linear or Sentry request with missing integration
9. existing automation task flow after environment selection or creation
10. clearly local request that stays in Kiro

## Known limitations

- this version does not run arbitrary `ona environment exec` commands
- this version does not write automation config with `ona automations update`
- this version does not create brand new automation tasks or services from natural language
- this version does not auto-open an editor after environment creation unless the user asks
- v1 does not require or configure an MCP server
- the power still relies on Kiro command approvals for both preflight checks and confirmed CLI actions

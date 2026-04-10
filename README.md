# Offload to Ona

`Offload to Ona` is a custom Kiro power that helps users decide when work should move from the IDE into Ona, checks local readiness, resolves the current repository to an Ona project, and launches the next step with the Ona CLI after explicit confirmation.

This version is intentionally scoped:

- automatic local detection when command execution is available
- project-first matching against existing Ona projects
- ranked project selection for repositories with many matching projects
- local Dev Container and Ona config preparation even when the Ona CLI is missing
- confirmed CLI-backed login, project creation, environment creation, one-off AI execution, and saved-automation start flows
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
- `ready_to_prepare_repo`
- `needs_ona_login`
- `needs_project_resolution`
- `needs_git_auth`
- `needs_integration`
- `ready_to_create_environment`
- `ready_to_start_ai_execution`
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
- `ona project list --limit 1000 -o json`
- `ona environment list -a -o json`

### Confirmed side-effect commands

These require explicit user confirmation before the power runs them:

- `ona login`
- `ona login --no-browser`
- `ona project create <repo-url> ...`
- `ona environment create <project-id> --dont-wait --set-as-context ...`
- `ona environment start <environment-id> --set-as-context`
- `ona ai automation execute - --environment-id <environment-id>`
- `ona ai automation start <automation-id> --project <project-id>`
- `ona automations task list -e <environment-id> -o json`
- `ona automations task start <task-ref> -e <environment-id> --dont-wait`

This version does **not** run arbitrary `ona environment exec` commands or write automation config into environments.

Command-template rule:

- use flags only when the specific command supports them
- do not assume `-o json` works on `ona environment create`
- when a command already prints the needed identifier directly, prefer the plain command over adding formatting flags

## Prompt-driven execution

For one-off requests like "run the entire test suite overnight in Ona", the power should:

1. resolve or select the Ona project
2. prefer the environment created earlier in the current Kiro session, otherwise create a fresh environment
3. preserve the user's original request as the AI prompt
4. run a one-off AI execution against that environment

The intended CLI primitive for this is:

```bash
cat <<'EOF' | ona ai automation execute - --environment-id <environment-id>
- agent:
    prompt: |
      <original user request>
EOF
```

This is different from:

- `ona ai automation start`, which starts a saved automation definition
- `ona automations task start`, which runs a predefined repo task from `.ona/automations.yaml`

The power should only use predefined repo tasks when the user explicitly asks for a known task.

Environment reuse policy for one-off runs:

- if this Kiro session already created an environment for the task, reuse that environment unless the user asks otherwise
- if this Kiro session did not create one, prefer creating a fresh environment over attaching to some unrelated running environment
- only reuse an older or externally created environment when the user explicitly asks to use that environment or names its ID

For one-off runs, that YAML should be treated as temporary execution input:

- prefer stdin so no file is created in the repo
- if stdin is inconvenient, use a temp file outside the repo and remove it after execution
- do not create `.ona/*.yaml` in the repo for a one-off prompt unless the user explicitly asks to keep it

For repeatable workflows, the power can suggest saving AI automation definitions as code under `.ona/` in the repository, then telling the user they can instantiate or reuse them with `ona ai automation create`.

## No-CLI setup behavior

If the Ona CLI is missing, the power should still help when the user wants to make the repository Ona-ready.

In that case, it should:

- switch from “launch Ona” to “prepare the repo”
- generate or update:
  - `.devcontainer/devcontainer.json`
  - `.ona/automations.yaml`
- use the canonical setup prompt from Ona's existing “Automated dev environment setup” workflow

This means a missing CLI should not block repository setup work.

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
- mutating Ona CLI flows should only be run after the user explicitly confirms the plan
- users do not need to grant broad wildcard trust for this version to work

Recommended approach:

- approve preflight commands as prompted
- after the power proposes a short multi-step plan, one user confirmation should cover the agreed flow unless something materially changes

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
3. fetch the full project set with `ona project list --limit 1000 -o json`
4. derive recent personal project usage from `ona environment list -a -o json` when possible
5. use the matching project when there is exactly one match
6. show all matches when there are up to 10
7. show a ranked top 5 plus a path to show all when there are more than 10
8. offer project creation only when there is no match and the repo appears Ona-ready

Ranking order:

1. exact repo match
2. recent personal usage inferred from environments
3. stronger project signals such as `usedBy.totalSubjects`, `automationsFilePath`, and `devcontainerFilePath`
4. project-name hints from the user request
5. clean, canonical names over obviously throwaway names
6. fallback alphabetical order

Name-quality heuristic:

- strongly downrank names that look temporary or unsafe such as `do-not-open`, `test`, `testing`, `delete`, `ignore`, or scratch-style labels
- do not downrank those names if the user explicitly asks for them
- if one candidate is clearly dominant by recency and usage, recommend it first instead of asking the user to sift through low-quality matches

The default minimum signal for automatic project creation is:

- `.devcontainer/devcontainer.json` exists

## Canonical setup prompt

For local repository setup, the power should use the prompt from Ona's existing “Automated dev environment setup” workflow rather than inventing a separate one-off prompt.

The source of truth for that workflow currently lives in Ona's product code and aligns with the public docs story:

- [Set up your first environment](https://ona.com/docs/ona/configuration/devcontainer/getting-started)
- [Ona docs source bundle](https://ona.com/docs/llms.txt)
- [Automations as code](https://ona.com/docs/ona/automations/automations-as-code)

The prompt instructs the agent to analyze the codebase, generate or update `devcontainer.json` and Ona automations, use the allowed docs sources, and avoid creating documentation files.

## Large project sets

This power is designed to work in orgs where many projects may point at the same repository.

Behavior expectations:

- it should never hard-truncate the match set to 3
- it should never print the raw org-wide `ona project list` output to the user as the selection UI
- it should show all matches when the set is small enough to scan
- it should show a ranked preview when the set is large
- it should support follow-ups like:
  - `show all`
  - `filter hosted`
  - `use 0198131b-296d-7c26-b1b6-1f6e3905174c`
- when one project is obviously dominant, it should recommend that candidate directly and explain why
- after project selection, it should keep environment reuse scoped to the current Kiro session unless the user explicitly opts into another environment

The project ID is always the escape hatch for power users.

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
9. `Show all matching projects for this repo.`
10. `Filter the matching projects to the hosted compute one.`
11. `Use project 0198131b-296d-7c26-b1b6-1f6e3905174c.`
12. `Set up this repository for Ona even if the Ona CLI is not installed.`
13. `Create the Dev Container and Ona automations config using the standard Ona setup prompt.`

## Suggested release flow

Use a GitHub-first rollout:

1. Validate locally with **Add power from Local Path**
2. Ask a few users to install from GitHub and test onboarding
3. Use GitHub tags or GitHub Releases for versioned updates
4. Consider a curated `kiro.dev/powers` listing later if adoption is strong

## Troubleshooting

### `ona` CLI is missing

The power should choose between two paths:

- open [app.ona.com](https://app.ona.com)
- sign in there
- install the CLI later if command-line flows are needed

Or, if the user wants repo preparation rather than direct launch:

- set up `.devcontainer/devcontainer.json`
- set up `.ona/automations.yaml`
- use the canonical Ona setup prompt

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

The power should not guess. It should:

- rank the matches
- show all matches when the list is short
- offer `show all` when the list is long
- allow `filter <text>` and `use <project-id>`

When ranking, it should prefer:

- projects you have actually used recently
- projects with stronger shared-usage signals
- projects with real Ona config attached
- canonical names like `Next` over throwaway names like `PD-DO-NOT-OPEN`

### The user asked for a one-off Ona run, but the power tried to start a repo task

That is the wrong execution path unless the user explicitly asked for a predefined task.

The power should:

- treat one-off long-running work as prompt-driven AI execution
- preserve the original user request as the prompt
- use `ona ai automation execute ... --environment-id <environment-id>`
- keep the generated YAML ephemeral via stdin or a temp file
- reserve `.ona/automations.yaml` task discovery for explicit task requests
- treat `ok`, `yes`, or equivalent approval as approval for the agreed create-and-run flow, not just the first command

### The power proposed the wrong Ona CLI command

The power should:

- validate command flags against the live CLI help before recommending them
- avoid unsupported formatting flags such as `-o json` on `ona environment create`
- fall back to the simplest known-good command form when only the environment ID is needed

### The power tried to reuse some other running environment automatically

That should not be the default.

The power should:

- reuse the environment it created earlier in the current Kiro session
- otherwise create a fresh environment for the current task
- only use a different existing environment when the user explicitly asks for it

### The user wants to keep a successful workflow for reuse

That is the point where the power should switch from ephemeral execution input to repo-stored automation-as-code.

The power should:

- explain that the one-off execution YAML was temporary
- suggest saving a reusable AI automation definition under `.ona/`
- make clear that `.ona/automations.yaml` is for repo tasks and services, while additional `.ona/*.yaml` files can hold reusable AI automation definitions
- only write or keep those files when the user explicitly wants a repeatable workflow

## Manual test cases

Validate these flows in Kiro:

1. authenticated user with a matching project
2. unauthenticated user who confirms `ona login`
3. no CLI installed
4. no CLI installed, but the user asks to prepare the repo for Ona
5. repo with multiple matching projects
6. repo with more than 10 matching projects
7. `show all` follow-up after a ranked preview
8. `filter <text>` follow-up that narrows the list
9. `use <project-id>` follow-up that bypasses ranking
10. repo with no matching project but a valid devcontainer
11. repo with no matching project and no devcontainer
12. PR-oriented request with missing Git auth
13. Linear or Sentry request with missing integration
14. existing automation task flow after environment selection or creation
15. clearly local request that stays in Kiro

## Known limitations

- this version does not run arbitrary `ona environment exec` commands
- this version does not write automation config with `ona automations update`
- this version does not create brand new automation tasks or services from natural language
- this version does not auto-open an editor after environment creation unless the user asks
- v1 does not require or configure an MCP server
- the power still relies on Kiro command approvals for both preflight checks and confirmed CLI actions

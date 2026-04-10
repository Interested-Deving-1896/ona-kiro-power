# Login And Readiness

Use this guide when the user wants to offload work to Ona and you need to determine what is ready.

## Readiness state machine

Work through these states in order:

1. Determine whether Ona is a fit for the task.
2. If yes, determine local CLI readiness if possible.
3. Resolve the current repository to an Ona project when possible.
4. Determine whether the task also requires Git provider authentication.
5. Determine whether the task also requires optional integrations.

Use exactly these states when reasoning:

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

## Detection behavior

When command execution is available:

1. Run `command -v ona`
2. Run `ona whoami -o json`
3. Run `git remote get-url origin`
4. Run `ona project list -o json`
5. Run `ona environment list -a -o json`

Interpretation:

- `command -v ona` fails and the user wants local repo setup -> `ready_to_prepare_repo`
- `command -v ona` fails and the user wants direct Ona execution -> `needs_cli`
- `command -v ona` succeeds and `ona whoami -o json` fails -> `needs_ona_login`
- local auth succeeds and the repo resolves to exactly one project -> `ready_to_create_environment`
- local auth succeeds and the repo resolves to multiple projects -> `needs_project_resolution`
- local auth succeeds and the repo resolves to no projects -> inspect repo readiness before offering project creation

After project resolution:

- if the user wants a one-off long-running outcome driven by their prompt -> `ready_to_start_ai_execution`
- if the user explicitly wants a saved automation -> use the saved automation path
- if the user explicitly wants a predefined repo task -> `ready_to_start_existing_task`

When command execution is not available:

- treat execution state as unavailable
- give browser-first guidance instead of claiming the power can run CLI-backed actions

## No-CLI local setup path

If the user wants to prepare the repository for Ona rather than launch Ona immediately:

- do not block on missing CLI
- offer a local setup workflow that generates or updates:
  - `.devcontainer/devcontainer.json`
  - `.ona/automations.yaml`
- use the canonical “Automated dev environment setup” prompt text from Ona's existing automation template
- treat this as a local repository-improvement flow, not an Ona-login flow

## Project resolution

Use a project-first strategy that scales to large numbers of matching projects.

Resolution algorithm:

1. derive the repository URL from `git remote get-url origin`
2. normalize SSH and HTTPS GitHub URL variants before matching
3. inspect `ona project list -o json`
4. inspect `ona environment list -a -o json`
5. match on repository metadata under the project initializer, not just project name

Observed project metadata can include:

- `initializer.specs[].git.remoteUri`
- `metadata.name`
- `id`

Observed environment metadata can include:

- `metadata.projectId`
- `metadata.lastStartedAt`
- `metadata.createdAt`

Matching rules:

- exactly one match -> use that project ID
- 2 to 10 matches -> show all matching projects
- more than 10 matches -> show a ranked top 5 plus a path to show all
- zero matches -> check `.devcontainer/devcontainer.json`

Ranking rules:

1. exact repository match
2. recent personal usage derived from environments when available
3. project-name hints from the user's wording when available
4. fallback alphabetical order

Recent usage heuristic:

- map recent environments to projects via `metadata.projectId`
- sort candidate projects by the newest available environment timestamp for the current user
- prefer `metadata.lastStartedAt` when available, otherwise `metadata.createdAt`

Do not label a project as "recent" unless that ranking really came from environment history.

If no project exists:

- if `.devcontainer/devcontainer.json` exists, offer project creation after confirmation
- if it does not exist, explain that the repo is not ready for automatic project creation

Interactive selection rules:

- allow `show all`
- allow `filter <text>`
- allow `use <project-id>`
- allow `use <project-name>` only when it narrows to a single project

Do not silently choose between multiple projects.

## Distinguish the setup layers

### Ona login

Use this when the user needs access to Ona itself.

Preferred guidance:

- open `https://app.ona.com`
- or run `ona login`
- or run `ona login --no-browser`
- or use `ona login --token <token>` with a PAT

PAT page:

- `https://app.ona.com/settings/personal-access-tokens`

### Git provider authentication

Use this when the user expects Ona to:

- clone a repository
- push commits
- create or help create a PR

Git auth page:

- `https://app.ona.com/settings/git-authentications`

Never describe this as "just log into Ona."

### Optional integrations

Use this only when the workflow depends on them:

- Linear for issue pickup or ticket context
- Sentry for issue investigation
- Notion for workspace documentation context

Integrations page:

- `https://app.ona.com/settings/org-integrations`

## Confirmation rules

All side-effecting CLI actions require user confirmation.

Supported confirmed actions:

- `ona login`
- `ona login --no-browser`
- `ona project create <repo-url> ...`
- `ona environment create <project-id> --dont-wait --set-as-context ...`
- `ona environment start <environment-id> --set-as-context`
- `ona ai automation execute - --environment-id <environment-id>`
- `ona ai automation start <automation-id> --project <project-id>`
- `ona automations task list -e <environment-id> -o json`
- `ona automations task start <task-ref> -e <environment-id> --dont-wait`

When a command is available:

- tell the user the exact command you intend to run
- explain the reason for running it
- wait for confirmation

Do not auto-run login or environment creation just because the user asked to use Ona.

## Required response structure

For any offload recommendation, include:

### `Readiness`

Use one sentence naming the current state and the most important consequence.

Examples:

- `Readiness`: Ready to create environment. The local Ona CLI is authenticated and this repository resolves to a single Ona project.
- `Readiness`: Ready to start AI execution. The local Ona CLI is authenticated, the project is selected, and I can run the user's prompt inside Ona after you confirm the exact command.
- `Readiness`: Needs Ona login. The local Ona CLI is present, but the current session is not authenticated.
- `Readiness`: Needs CLI. I could not find the Ona CLI locally, so I cannot launch anything directly from this power yet.
- `Readiness`: Ready to prepare the repo. I could not find the Ona CLI locally, but I can still set up the Dev Container and Ona configuration files in this repository.
- `Readiness`: Needs project resolution. I found multiple matching Ona projects, ranked the best candidates, and can show or filter the full list before creating anything.

### `Additional setup needed`

Only list blocking or near-blocking items relevant to the exact task.

Examples:

- Git provider auth for clone and PR access
- Linear integration for ticket context
- Sentry integration for issue context
- Project creation because no Ona project currently matches this repository

Do not list all possible Ona setup steps on every response.

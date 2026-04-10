# Execution Flow

Use this guide when the user wants the power to actually run Ona CLI commands.

## Execution model

This power is a launcher, not a full orchestration agent.

Its supported execution path is:

1. run preflight checks
2. resolve the current repo to an Ona project if possible
3. rank and narrow candidate projects if there are multiple matches
4. explain readiness and the exact next command
5. ask for confirmation
6. run the confirmed command
7. report the result and stop unless the user explicitly asks for the next step

It also supports a local repository-preparation path:

1. detect that the user wants Ona-ready setup rather than direct Ona launch
2. if the Ona CLI is missing, switch to local setup instead of blocking
3. use the canonical setup prompt to generate Dev Container and Ona config files
4. validate locally when practical
5. report what was prepared and what still requires Ona

## Safe preflight commands

These are safe to run automatically when Kiro allows shell commands:

- `command -v ona`
- `ona whoami -o json`
- `git remote get-url origin`
- `ona project list --limit 1000 -o json`
- `ona environment list -a -o json`

These commands should be enough to detect:

- whether the CLI exists
- whether the user is logged in
- what repository is currently in scope
- whether an Ona project already exists for that repo
- which matching projects the user has recently used

## Confirmed side-effect commands

Only run these after the user explicitly confirms:

- `ona login`
- `ona login --no-browser`
- `ona project create <repo-url> --name <derived-name> ...`
- `ona environment create <project-id> --dont-wait --set-as-context --name <derived-name>`
- `ona environment start <environment-id> --set-as-context`
- `ona ai automation execute - --environment-id <environment-id>`
- `ona ai automation start <automation-id> --project <project-id>`
- `ona automations task list -e <environment-id> -o json`
- `ona automations task start <task-ref> -e <environment-id> --dont-wait`

## Local setup flow

Use this flow when the user wants the repository prepared for Ona and the CLI is missing or not required yet.

Expected outputs:

- `.devcontainer/devcontainer.json`
- `.ona/automations.yaml`
- any minimal supporting files needed to make the setup work

Use the canonical setup prompt from Ona's “Automated dev environment setup” workflow rather than inventing a shorter ad hoc prompt.

Local validation should prefer:

- checking the generated files for obvious completeness and consistency
- project-specific sanity checks from the repository
- Ona-specific CLI validation only when the CLI is actually available

## Environment creation flow

When the current repo resolves to exactly one project:

1. explain that the project match was found
2. offer the exact `ona environment create` command
3. ask for confirmation
4. run it after confirmation
5. report the environment ID or creation result

Default flags:

- `--dont-wait`
- `--set-as-context`
- `--name <derived-name>`

Do not automatically open the environment in an editor unless the user asks.

## One-off AI execution flow

Use this flow when the user wants Ona to take a goal or prompt and work on it, such as:

- "run the entire test suite overnight in Ona"
- "investigate this bug in Ona"
- "use Ona to fix this and raise a draft PR"

Suggested flow:

1. resolve or select the project
2. create a fresh environment or start a chosen stopped environment
3. preserve the user's original request as the prompt payload
4. offer the exact `ona ai automation execute` command before running it
5. execute a one-off steps spec with an `agent.prompt` step against the environment
6. report the execution result or identifier and how to monitor it

Preferred command shape:

```bash
cat <<'EOF' | ona ai automation execute - --environment-id <environment-id>
- agent:
    prompt: |
      <original user request>
EOF
```

Rules:

- do not replace the original user request with a repo task name
- prefer stdin so the one-off YAML never lands in the repository
- if stdin is inconvenient, write a temp file outside the repo and delete it after execution
- do not leave one-off YAML behind in the repo unless the user explicitly wants to keep it
- do not inspect `.ona/automations.yaml` unless the user explicitly asked for a predefined repo task
- do not classify a one-off overnight request as recurring automation just because it is long-running
- if the environment was created with `--dont-wait`, be prepared to start or wait for it before launching the AI execution if needed

## Multiple project match flow

When the repo matches multiple projects:

1. rank matches by:
   - exact repo match
   - recent personal usage from environments
   - stronger project signals like `usedBy.totalSubjects`, `automationsFilePath`, and `devcontainerFilePath`
   - project-name hints from the user's wording
   - clean names before throwaway names
   - fallback name order
2. if there are 2 to 10 matches, show all of them
3. if there are more than 10 matches, show the top 5 and state how many remain
4. support user follow-ups:
   - `show all`
   - `filter <text>`
   - `use <project-id>`
   - `use <project-name>`
5. only proceed once a single project is selected

Do not hard-truncate to 3 candidates.

Do not present the raw output of `ona project list` as the selection experience. Filter and rank in the shell first, then present only the repo-matching candidates.

If one project clearly dominates, such as:

- very recent personal usage
- much higher shared usage
- real Ona config present
- a canonical repo name like `Next`

then recommend that candidate first and explain why, while still allowing the user to override it.

## Project creation flow

Only offer project creation when:

- the current repo does not already match a project
- `.devcontainer/devcontainer.json` exists

Suggested flow:

1. explain that no matching project exists
2. explain that the repo appears ready because a devcontainer exists
3. offer the exact `ona project create` command
4. ask for confirmation
5. if project creation succeeds, offer environment creation as the next step

If the repo lacks a devcontainer, stop and explain why project creation is not being offered automatically.

## Existing automation task flow

Only use this when the user explicitly wants to run an existing automation task from the repo configuration.

Suggested flow:

1. ensure there is a selected or newly created environment
2. offer to run `ona automations task list -e <environment-id> -o json`
3. show concise task choices
4. ask which task to start
5. offer the exact `ona automations task start <task-ref> -e <environment-id> --dont-wait` command
6. ask for confirmation before starting it

Do not invent new automation task definitions in this flow.

## Saved automation flow

Use this flow when the user wants to run a saved Ona AI automation, not just a one-off prompt.

Suggested flow:

1. identify the saved automation ID
2. confirm the project or repository context
3. offer the exact `ona ai automation start <automation-id> --project <project-id>` command
4. ask for confirmation before running it

This is different from:

- one-off prompt-driven execution via `ona ai automation execute`
- predefined repo task execution via `ona automations task start`

When a one-off run proves useful and the user wants to repeat it:

1. explain that the original one-off YAML was ephemeral
2. offer to save a reusable AI automation definition under `.ona/` in the repository
3. tell the user that repo-stored AI automation files can be instantiated or updated with `ona ai automation create` and related commands
4. keep `.ona/automations.yaml` reserved for repo tasks and services, not for every one-off prompt

## Reporting rules

After a command runs:

- report whether it succeeded
- include the most useful identifier, such as project ID, environment ID, or task execution ID
- tell the user the next available action

If a command fails:

- summarize the failure clearly
- do not claim that Ona itself is broken unless the error shows that
- fall back to the structured response contract and the shortest recovery path

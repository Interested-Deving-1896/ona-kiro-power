# Login And Readiness

Use this guide when the user wants to offload work to Ona and you need to determine what is ready.

## Readiness state machine

Work through these states in order:

1. Determine whether Ona is a fit for the task.
2. If yes, determine local CLI readiness if possible.
3. Determine whether the task also requires Git provider authentication.
4. Determine whether the task also requires optional integrations.

Use exactly these states when reasoning:

- `ready`
- `needs_ona_login`
- `unknown_auth_state`
- `needs_git_auth`
- `needs_integration`

## Detection behavior

When command execution is available:

1. Run `command -v ona`
2. Run `ona whoami`

Interpretation:

- both succeed -> `ready`
- `command -v ona` succeeds and `ona whoami` fails -> `needs_ona_login`
- `command -v ona` fails -> `unknown_auth_state`

When command execution is not available:

- use `unknown_auth_state`
- give browser-first guidance instead of blocking

## Distinguish the setup layers

### Ona login

Use this when the user needs access to Ona itself.

Preferred guidance:

- open `https://app.ona.com`
- or run `ona login`
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

## Required response structure

For any offload recommendation, include:

### `Readiness`

Use one sentence naming the current state and the most important consequence.

Examples:

- `Readiness`: Ready. The local Ona CLI is available and already authenticated.
- `Readiness`: Needs Ona login. The local Ona CLI is present, but the current session is not authenticated.
- `Readiness`: Unknown auth state. I could not verify local CLI access, so the safest path is to sign in through the browser first.

### `Additional setup needed`

Only list blocking or near-blocking items relevant to the exact task.

Examples:

- Git provider auth for clone and PR access
- Linear integration for ticket context
- Sentry integration for issue context

Do not list all possible Ona setup steps on every response.

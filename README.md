# Offload to Ona

`Offload to Ona` is a custom Kiro power that helps users decide when work should move from the IDE into Ona, checks basic readiness, and prepares a clean handoff.

This v1 power is intentionally lightweight:

- no direct environment creation
- no direct automation creation
- no MCP server requirement
- optional local checks for `command -v ona`, `ona version`, and `ona whoami`
- browser-first fallback when CLI checks are unavailable

## What it does

The power is designed for requests like:

- "Run this overnight in Ona"
- "This needs private network access. Should we move it to Ona?"
- "Turn this cleanup into an automation"
- "Pick up this Linear ticket in Ona"
- "Use Ona to fix this and raise a draft PR"

For matching requests, it classifies the task into one of four outcomes:

- `stay-local`
- `offload-agent`
- `offload-automation`
- `not-ready-yet`

For every offload recommendation, it returns:

- `Why Ona`
- `Recommended path`
- `Readiness`
- `Next step`
- `Ona prompt`
- `Additional setup needed`

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

## Login and setup expectations

This power treats readiness as three separate concerns.

### 1. Ona login

Preferred paths:

- browser: [app.ona.com](https://app.ona.com)
- CLI: `ona login`
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

## Example prompts

These are good prompts to verify activation and response quality:

1. `Run this test suite overnight in Ona and open a draft PR if it passes.`
2. `This bug needs access to our internal staging network. Should we offload it to Ona?`
3. `Turn this dependency cleanup into a nightly Ona automation.`
4. `Use Ona to pick up my Linear ticket and prepare the implementation plan.`
5. `Should I keep this frontend tweak in Kiro or move it to Ona?`
6. `I want Ona to investigate this Sentry issue and propose a fix.`

## Suggested release flow

Use a GitHub-first rollout:

1. Validate locally with **Add power from Local Path**
2. Ask a few users to install from GitHub and test onboarding
3. Use GitHub tags or GitHub Releases for versioned updates
4. Consider a curated `kiro.dev/powers` listing later if adoption is strong

## Troubleshooting

### `ona` CLI is missing

The power should fall back to browser-first guidance:

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

### The user is logged into Ona but still cannot push or open PRs

The likely gap is Git provider authentication, not Ona login. Direct the user to:

- [app.ona.com/settings/git-authentications](https://app.ona.com/settings/git-authentications)

### The task mentions Linear, Sentry, or Notion but the workflow is blocked

The likely gap is an optional integration. The power should call that out separately instead of presenting it as an Ona login problem.

## Known limitations

- v1 does not create Ona environments directly
- v1 does not create Ona Automations directly
- v1 does not require or configure an MCP server
- v1 relies on Kiro being able to use local command checks if available; otherwise it falls back to browser guidance

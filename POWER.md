---
name: "ona"
displayName: "Offload to Ona"
description: "Decide when work belongs in Ona, check readiness, and hand off long-running, secure, or repeatable tasks with a ready-to-use Ona prompt."
keywords: ["ona", "gitpod", "background agent", "run overnight", "automation", "secure environment", "draft pr", "pull request", "linear", "sentry", "notion", "internal access", "long-running", "offload", "agent", "repeatable work", "recurring task", "vpc", "private network"]
---

# Onboarding

## Step 1: Understand when this power should help

Use this power when the user is deciding whether to keep work in Kiro or move it into Ona.

This power is a good fit when the user wants to:

- run something overnight or in the background
- move work into a secure or isolated environment
- use Ona for long-running code, test, or refactor work
- turn a repeated workflow into an Ona Automation
- use Linear, Sentry, or similar context inside Ona
- prepare a task that should end in a draft PR

This power is not a good fit when the user only needs:

- a quick local edit
- tight interactive iteration in the current IDE
- small exploratory debugging that benefits from staying local

## Step 2: Classify the request before suggesting Ona

Classify the request into exactly one of these outcomes:

- `stay-local`
- `offload-agent`
- `offload-automation`
- `not-ready-yet`

Default to:

- `offload-agent` for one-off long-running work
- `offload-automation` for recurring, scheduled, or multi-repository work
- `not-ready-yet` when Ona is a fit but login, Git auth, or integrations are likely missing

## Step 3: Check local Ona readiness if command execution is available

When the environment allows local command checks, run these checks in order:

1. `command -v ona`
2. `ona whoami`

Interpret them as follows:

- `command -v ona` succeeds and `ona whoami` succeeds: state is `ready`
- `command -v ona` succeeds and `ona whoami` fails with authentication-related output: state is `needs_ona_login`
- `command -v ona` fails or command execution is unavailable: state is `unknown_auth_state`

Do not run side-effecting Ona commands during detection.

## Step 4: Keep Ona login, Git auth, and integrations separate

Treat these as distinct readiness layers:

1. **Ona login**
   - Browser path: `https://app.ona.com`
   - CLI path: `ona login`
   - PAT fallback: `ona login --token <token>`

2. **Git provider authentication**
   - Needed for cloning, pushing, and PR work
   - Do not imply this is covered by Ona login alone
   - Point users to `https://app.ona.com/settings/git-authentications` when relevant

3. **Optional integrations**
   - Linear, Sentry, Notion, and similar tools are only required when the user's requested workflow depends on them

## Step 5: Use the structured response contract

For every `offload-agent`, `offload-automation`, or `not-ready-yet` response, use these exact sections in order:

1. `Why Ona`
2. `Recommended path`
3. `Readiness`
4. `Next step`
5. `Ona prompt`
6. `Additional setup needed`

Keep the response practical and compact. The user should be able to copy the Ona prompt directly into Ona.

## Step 6: Use browser-first fallback when checks are unavailable

If you cannot reliably determine CLI auth state, do not block. Fall back to browser guidance:

- tell the user to open `https://app.ona.com`
- tell them to sign in there
- mention CLI login as optional follow-up for users who want terminal-based workflows

# When To Load Steering Files

- Deciding whether to keep work local or offload it to Ona -> `steering/delegate-to-ona.md`
- Determining readiness, login state, Git auth, or integration gaps -> `steering/login-and-readiness.md`
- Turning a repeated task into an Ona Automation handoff -> `steering/automation-handoff.md`
- Checking whether a repository is ready for Ona-based work -> `steering/repo-readiness.md`

# Core Rules

- This v1 power is guide-and-handoff only. Do not claim to create Ona environments or automations directly.
- Prefer Ona when the task is long-running, secure, internal-network-dependent, or repeatable.
- Prefer staying local when the user benefits more from tight IDE iteration.
- If the user asks for PR-oriented work, mention Git provider authentication separately from Ona login.
- If the user asks for Linear, Sentry, or Notion context, mention those integrations only if they are relevant to the requested task.

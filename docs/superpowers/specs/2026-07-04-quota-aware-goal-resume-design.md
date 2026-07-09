---
summary: "History and design notes for quota-aware resumption of paused agent goals."
read_when:
  - Designing auto-resume after Codex/Claude quota resets
  - Revisiting notification external actions, Shortcuts, or webhooks
  - Turning paused agent goals into durable work
---

# Quota-Aware Goal Resume

**Status:** personal history note and proposal, not approved
**Date:** 2026-07-04
**Related local branches:** `notification-model`, `notification-shortcuts`, `notification-webhooks`,
`quota-notifications-migration`
**Related PRs:** [#801](https://github.com/steipete/CodexBar/pull/801),
[#1010](https://github.com/steipete/CodexBar/pull/1010)

## Problem

Long-running agent work can consume the current session quota, stop for several hours, and then fail to continue when
the quota window opens again. The user experience is broken in two places:

1. The agent or CLI often knows the reset time, but the interrupted work is left for the user to rediscover manually.
2. When the user later types `continue`, the session may have lost the exact task intent, current file context, or
   safe next action.

The target behavior is not merely "send continue later." The target behavior is: when a quota pause happens, preserve
the user's active goal as durable work, wait for the quota to recover, and resume the safest available continuation path
without losing intent or duplicating side effects.

## History from prior CodexBar PRs

PR [#801](https://github.com/steipete/CodexBar/pull/801) attempted to add notification delivery options:

- local macOS notifications
- configurable sounds
- Apple Shortcuts
- webhook URLs
- custom URL schemes
- a Notifications preferences pane
- migration of existing session quota depleted/restored notifications

The motivation was correct: CodexBar can be the quota event source, and external automation can decide what to do when
quota is restored, including resuming a paused agent/task.

The PR was closed because the branch shape was too broad and stale against current `main`, not because the automation
idea was rejected. The maintainer feedback pointed to a smaller series:

1. isolated notification event/delivery model with tests
2. one external action type first, likely Apple Shortcuts
3. webhook delivery separately, with URL validation and privacy notes
4. move existing quota notifications only after the delivery seam is proven

One concrete bug was also found: logging full webhook URLs can leak bearer-style secrets in paths or query strings. Any
future webhook work must log only scheme/host or a redacted URL.

PR [#1010](https://github.com/steipete/CodexBar/pull/1010) reopened the first layer as a smaller draft:

- typed notification delivery model
- event-specific delivery settings
- local notification routing only
- no preferences UI, Shortcuts, webhooks, or quota migration

It was also closed as stale because only the model layer was published and the product-complete slice was never finished.

## Prior art found on GitHub

### Direct auto-resume tools

- [terryso/claude-auto-resume](https://github.com/terryso/claude-auto-resume): shell utility that detects Claude usage
  limits, waits until the timestamp, and invokes `claude -c -p "continue"` or a custom command. Useful proof of demand,
  but it relies on unattended permission bypass and prompt-level continuation.
- [cheapestinference/claude-auto-retry](https://github.com/cheapestinference/claude-auto-retry): tmux-based monitor that
  detects rate-limit messages, parses reset time, verifies Claude is still foreground, and sends `continue`.
- [henryaj/autoclaude](https://github.com/henryaj/autoclaude): TUI that monitors tmux panes and auto-continues selected
  Claude Code panes after reset.
- [riazarbi/way](https://github.com/riazarbi/way): shell orchestration wrapper with `Claude AI usage limit reached`
  timestamp parsing and retry.
- [patou2024/orchclaude](https://github.com/patou2024/orchclaude): roadmap explicitly calls for crash recovery,
  session files, usage-limit detection, `autowait`, and `autoschedule`.

These tools prove a simple implementation can work, but they are mostly terminal-observation wrappers. They do not solve
durable intent capture, idempotency, provider-neutral scheduling, or cross-session safety.

### Native feature requests

- [openai/codex#21073](https://github.com/openai/codex/issues/21073): asks Codex CLI to sleep until `resets_at` and
  rerun the same turn.
- [openai/codex#8310](https://github.com/openai/codex/issues/8310): reports that resume after rate limit can lose task
  intent and continue in the wrong context.
- [anthropics/claude-code#35744](https://github.com/anthropics/claude-code/issues/35744): open request for native
  auto-continue after subscription rate-limit reset.
- [anthropics/claude-code#26775](https://github.com/anthropics/claude-code/issues/26775),
  [#36320](https://github.com/anthropics/claude-code/issues/36320), and
  [#38263](https://github.com/anthropics/claude-code/issues/38263): duplicate requests around automatic waiting,
  reset parsing, and `--continue`.
- [ruvnet/ruflo#133](https://github.com/ruvnet/ruflo/issues/133): closed request for `claude-flow` automatic wait and
  resume on usage limit.

The repeated issue pattern is clear: users do not only want quota notifications; they want the work itself to survive the
quota interruption.

### Durable agent patterns

- [OpenHands pause/resume](https://docs.openhands.dev/sdk/guides/convo-pause-and-resume): conversation execution can be
  paused and later resumed without losing state.
- [LangGraph checkpointers](https://docs.langchain.com/oss/javascript/langgraph/checkpointers): state is saved by thread
  at graph supersteps, enabling fault-tolerant execution and human-in-the-loop resume.
- [Microsoft Agent Framework Durable Task extension](https://learn.microsoft.com/en-us/azure/durable-task/sdks/durable-agents-microsoft-agent-framework):
  persistent sessions, automatic checkpointing, long waits, and recovery are treated as durable execution.
- [ccusage/ccusage](https://github.com/ccusage/ccusage): parses Claude local usage data and includes a robust parser for
  the `Claude AI usage limit reached|<timestamp>` line.

The durable-agent pattern is the better architectural target: persist a resumable work item and explicit continuation
state, then schedule wakeup. A terminal wrapper can be a compatibility adapter, not the core design.

### Coordination risk

The rate-limit problem gets worse with multiple agents. A useful external writeup,
[9 AI Agents, One API Quota](https://www.tamirdresher.com/blog/2026/03/21/rate-limiting-multi-agent), calls out
thundering-herd retry, priority inversion, and cascade amplification. Any serious auto-resume design needs jitter,
priority, max concurrency, and a shared rate-governor concept so every paused task does not resume at the same second.

## Design direction

CodexBar should not try to own every agent runtime. It should become a reliable quota event source and local resume
broker:

1. Detect quota-depleted and quota-restored transitions from provider snapshots.
2. Let a trusted local adapter register a durable resume intent.
3. On restoration, evaluate safety gates and trigger the adapter.
4. Record what happened, without logging secrets or private account data.

This splits the system into three parts:

### 1. Quota event source

CodexBar already has provider refreshes, rate windows, depleted/restored transitions, reset times, account scoping, and
notifications. Keep this layer provider-owned and privacy-preserving.

Required events:

- `quota.depleted`
- `quota.restored`
- `quota.resetScheduled`
- later: `quota.atRisk` from predictive pace warnings

Events must include only stable internal identifiers, provider/window identity, reset timestamp, and redaction-safe
display copy.

### 2. Resume intent store

A resume intent is a small durable record registered by an adapter, not inferred from random terminal text.

Minimum fields:

- `id`
- `createdAt`
- `provider`
- `window`
- `workspacePath`
- `agentKind` (`codex-cli`, `claude-code`, `tmux-pane`, `custom-command`, etc.)
- `resumeCommand` or adapter-specific opaque target
- `lastKnownGoal`
- `checkpointSummaryPath` or `checkpointText`
- `blockedBecause` (`quota-depleted`, `rate-limited`, `manual-pause`)
- `notBefore`
- `expiresAt`
- `maxResumeAttempts`
- `requiresUserApproval`
- `idempotencyKey`

The important part is the checkpoint summary. A blind `continue` is acceptable only when the original session is still
alive and known to be waiting at the rate-limit screen. Otherwise the adapter must resume with a compact handoff:

- what the user asked for
- what was completed
- what files were touched
- what commands/tests ran
- exact next safe step
- known blockers

### 3. Resume scheduler and adapters

The scheduler should support both in-process waits and external wakeups:

- in-process timer while CodexBar is running
- launchd/Shortcuts/webhook/custom URL as later delivery adapters
- jittered wakeup after reset to avoid retry storms
- max attempts and exponential backoff for transient failures
- optional "confirm before resume" mode
- a dry-run/audit view listing pending resume intents

Adapters translate a resume event into action:

- `codex-cli`: run a safe `codex resume/continue` command if such a stable surface exists.
- `claude-code`: foreground tmux pane continuation only if the pane still matches the expected process/session.
- `custom-command`: execute a user-supplied command only when explicitly configured.
- `Apple Shortcut`: first external-action candidate because it delegates side effects to user-owned automation.
- `webhook`: later, with strict URL redaction, secret handling, and method/body constraints.

## Safety rules

- Default off. No automatic resume for existing users.
- Never auto-send into an unknown active terminal.
- Prefer registered intents over terminal scraping.
- Require user approval for commands that can mutate files unless the user opted into unattended resume.
- Store and show pending intents; allow cancel.
- Do not log raw webhook URLs, bearer tokens, cookies, prompts containing secrets, or account identities.
- Add reset-time margin and jitter; never wake exactly at `resetsAt` for every intent.
- If multiple intents share a provider/window, resume by priority and concurrency limits.
- If a restored check fails or the provider is still depleted, keep the intent pending with bounded retry.
- If context is stale, resume by creating a new session with the checkpoint summary instead of sending bare `continue`.

## Recommended PR series

### PR 1: Current-main notification event model

Rebuild the `notification-model` idea on current `main`, but keep it small:

- typed event identity
- delivery settings model
- local notification adapter
- tests for session quota depleted/restored
- no Preferences UI
- no Shortcuts
- no webhooks
- no migration of existing behavior unless fully covered

### PR 2: Apple Shortcuts action for quota-restored

Add one external action first:

- Shortcut name setting
- only for quota-restored event initially
- explicit user opt-in
- no arbitrary shell command
- focused tests around delivery invocation and privacy

This gives users an immediate bridge to iPhone/Watch/Home Assistant/Pushcut/Raycast without making CodexBar a general
automation runner.

### PR 3: Webhook action

Only after PR 2 lands:

- validate URL scheme
- redact URL in logs
- no secrets in diagnostics
- bounded timeout
- explicit retry policy
- tests for path/query redaction

### PR 4: Resume intent store

Add local durable pending work:

- `ResumeIntent` model and JSON store
- dry-run/status UI or CLI diagnostics
- pruning and cancel behavior
- no automatic execution yet

### PR 5: Quota-aware resume broker

Connect restored quota events to pending intents:

- restored event wakes eligible intents
- safety gates and jitter
- one adapter first, likely Apple Shortcut or custom URL
- audit trail of attempted/resumed/failed/cancelled

## Acceptance criteria for the eventual full solution

- Hitting a session quota can register a durable resume intent with a reset time and checkpoint summary.
- CodexBar can show pending paused work and let the user cancel it.
- When quota restores, CodexBar resumes only eligible work and records the result.
- The same quota window cannot trigger duplicate resumes for the same idempotency key.
- Multiple pending tasks do not thundering-herd into the provider.
- If the original terminal/session is gone, the adapter either uses a checkpoint summary or refuses safely.
- Full webhook/custom-command targets never leak into logs.
- Tests cover provider/account/window isolation, reset identity, retries, cancellation, and privacy.

## Bottom line

The old PR was pointing at the right user problem but tried to ship too much at once. The universal version should be:

1. small current-main event model,
2. one safe external action,
3. durable resume intents,
4. quota-restored scheduler,
5. adapter-specific resume behavior.

That keeps CodexBar as the provider quota authority while leaving agent-specific continuation to narrow, testable
adapters.

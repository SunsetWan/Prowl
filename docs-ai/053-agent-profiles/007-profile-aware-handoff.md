# 053.007 — Profile-aware handoff

| | |
| --- | --- |
| **Date** | 2026-08-01 |
| **Status** | Planned |
| **Primary PRs** | TBD |
| **Related** | [053 plan](000-plan.md), [053.006](006-launch-scoped-environment.md), [047.004](../047-cross-agent-handoff/004-inline-handoff-redesign.md), [047.005](../047-cross-agent-handoff/005-hud-request-ownership.md), [049 Agents HUD](../049-agents-toolbar-entry/000-plan.md), [048 runtime adapters](../048-agent-runtime-adapters/000-plan.md), [handoff manual](../../docs/components/handoff.md) |

## Context

The Hand Off HUD and `prowl handoff to` currently identify a receiver only by runtime token.
The CLI handler rebuilds a small inherited `AgentLaunchConfiguration`, while the HUD fallback
independently renders a runtime invocation and creates a tab. Neither route can select a Prowl
Agent Profile, so they lose the profile's model, effort, extra arguments, launch-scoped environment,
Dedicated Home, account, and surface identity.

This follow-up closes the seam reserved by 053. A native Codex Config Profile remains ordinary
profile configuration: users can already put `-p work` in a Prowl Agent Profile's Extra Arguments.
This wave does not add a Codex-specific field or change the `AgentProfile` persistence schema.

## Goals

- Let the HUD and CLI select an enabled Prowl Agent Profile by stable UUID.
- Preserve Runtime Default Claude Code/Codex targets and their existing inheritance behavior.
- Make inline CLI and HUD fallback handoffs execute the same complete profile launch plan.
- Keep artifact, request-ownership, background-launch, notification, and HUD-focus semantics honest
  across preflight and receiver-launch failures.
- Keep profile configuration and secrets out of handoff requests, artifacts, logs, and responses.

### Non-goals

- A dedicated Codex Config Profile editor field; `-p <name>` stays in Extra Arguments.
- Propagating the outgoing pane's Profile, Dedicated Home, environment, or account into source
  resume/fork collection.
- Removing runtime-default targets or changing their model/unrestricted inheritance.
- Honoring a Receiving Profile's manual-launch `Open In` placement during handoff.
- Detecting whether the launched CLI remains healthy after its terminal surface is created.
- Adding automatic HUD timeouts, Profile import/export, or more runtimes.

## Contract

### Receiving Target

Introduce one runtime-neutral domain value with two cases:

```swift
enum HandoffReceivingTarget {
  case runtimeDefault(DetectedAgent)
  case profile(AgentProfile.ID)
}
```

An enabled Profile is shown as a Receiving Profile. Runtime Default remains the compatibility path.
The HUD orders targets as Recommended Profile, remaining enabled Profiles in Settings order, then
Runtime Defaults, then Brief Only. With no enabled Profiles, its target list behaves as it does now.

Profile names are presentation only. The request carries the UUID; runtime, display name, and launch
configuration are resolved from the latest persisted Profile after briefing collection and before
artifact commit, then frozen in memory for that execution. A missing or disabled Profile, or a launch
planning error, fails before any handoff artifact is mutated.

### CLI and wire format

Keep the existing form and add one mutually exclusive Profile form:

```bash
prowl handoff to codex [source] [options]
prowl handoff to --agent-profile-id <UUID> [options]
prowl handoff to --agent-profile-id <UUID> --pane <pane-id> [options]
```

- The runtime positional argument and `--agent-profile-id` are exactly-one.
- Runtime handoffs retain the optional positional source for compatibility.
- Profile handoffs accept no positional source; an explicit source uses `--pane`, `--tab`, or
  `--worktree`. With no selector, caller-pane self-handoff resolution remains unchanged.
- CLI parsing rejects both, neither, and malformed UUID cases before transport; the app handler
  repeats the invariant for direct or older clients.
- `HandoffInput` adds optional `to_profile_id`. Successful payloads add optional
  `to_profile_id` and frozen `to_profile_name` while keeping `to_agent` as the resolved runtime.
  These additive fields remain in `prowl.cli.handoff.v2`.
- `--no-launch` still resolves an existing, enabled Profile and records its frozen identity, but it
  does not compile/provision a launch, create a surface, or update Last Launched Profile.

### HUD request ownership and completion

Bind each HUD request UUID to its source pane plus exact operation: checkpoint, Runtime Default
handoff, or Profile handoff. A handoff operation also requires `launch == true`. The injected command
uses the selected Profile UUID and an explicit source-pane selector; it never embeds the Profile name.

Registry claim distinguishes claimed, target mismatch, superseded, duplicate, and unknown requests.
A mismatch returns a stable CLI error before briefing or artifact work and does not consume the
pending request, so the source agent may correct and retry. A fallback must still atomically supersede
the pending inline request before collecting ownership of the transition.

A mismatch is deliberately a retryable, non-terminal HUD event: the HUD keeps waiting and retains its
existing Fork Briefing, Context Only, and Cancel exits. Exactly-once terminal completion begins only
after a matching claim; this wave does not add an automatic timeout.

Once a matching request is claimed, the handler emits exactly one typed completion: success with
not-applicable/skipped/launched disposition, or failure with an `artifactsReady` flag. This lets the HUD
terminate on Profile/preflight failure and distinguish it from a post-commit receiver-launch failure;
the current success-only completion must not leave a claimed request waiting forever.

### Execution and failure boundary

The Profile path executes in this order:

```text
claim matching HUD request, if any
→ collect/validate briefing without artifact writes
→ resolve latest enabled Profile and compile prompted launch plan
→ commit transition artifacts
→ launch through the shared Profile surface boundary
→ append transition log and publish response/completion
```

The Receiving Profile is authoritative for model, effort, execution mode, Extra Arguments,
launch-scoped environment, Dedicated Home, account, and identity. It never inherits launch settings
from the outgoing agent. Runtime Default keeps the current explicit-model and observed-unrestricted
inheritance.

| Failure point | Required outcome |
| --- | --- |
| Invalid target, missing/disabled Profile, or plan failure | Error before artifact mutation; no launch-memory update |
| Artifact commit failure | Error; no receiver launch; do not claim a complete artifact set |
| Dedicated Home or surface creation failure after commit | Retain artifacts/archive, append `launch=failed`, report “progress saved, receiver not launched,” no launch-memory update |
| `--no-launch` | Successful archive-only result with `launch=skipped`; no provision, surface, or launch-memory update |
| Surface created | Record exact pane, notify, and emit the existing Profile launch-success event |

### Shared Profile launch boundary

Extend `AgentProfileLaunchPlanner.plan` with an `AgentStartIntent` parameter defaulting to
`.interactive`; handoff passes `.prompt(kickoffPrompt)`. The adapter therefore keeps Extra Arguments
such as Codex `-p work` before the positional prompt and remains the sole quoting authority.

Add a handoff launch context and typed result to the existing Profile terminal path. The handoff
context always creates a new background tab at the handoff root, ignores Profile tab/split placement,
preserves launch-scoped environment and Dedicated Home provisioning, and records Profile identity.
The result returns the exact tab and surface IDs without a post-creation resolver lookup.

`WorktreeTerminalState` and `WorktreeTerminalManager` stay synchronous on `@MainActor`.
`TerminalClient` exposes a synchronous main-actor Profile-launch closure so CLI and fallback callers
receive the typed result; they must not infer success from the fire-and-forget command path. Artifact
work may stay detached, but launch crosses back to the main actor.

`WorktreeTerminalManager` remains the one executor and emits the existing success/failure events.
Only a created surface updates per-repository Last Launched Profile through
`AppFeature+TerminalEvents`. Ordinary Profile launches retain their configured placement.

Core and CLI launches never focus the receiver. If the initiating HUD still exists and is waiting for
that exact completion, it focuses the returned pane after success. A dismissed HUD and an ordinary CLI
handoff leave the receiver in the background; launch failure leaves the user on the source pane.

### Output and privacy

The response and internal completion may expose only resolved runtime, Profile UUID, and frozen
single-line display name. The append-only log adds those same optional fields and sanitizes Profile
names against newline/quote injection. Archive filenames continue to use the runtime token. Extra
Arguments, environment names/values, carrier values, Dedicated Home paths, and credentials never enter
the handoff request, response, artifact, or log.

## Implementation plan

1. Add the exclusive CLI option, optional wire/payload fields, text rendering, and request-mismatch
   error in `ProwlCLI/Commands/HandoffCommand.swift`, `ProwlCLI/Output/OutputRenderer.swift`,
   `supacode/CLIService/Shared/InputModels.swift`, `supacode/CLIService/Shared/HandoffCommandPayload.swift`,
   and `supacode/CLIService/Shared/ErrorCodes.swift`.
2. Add Receiving Target/request-expectation types and exact claim semantics in
   `supacode/Domain/Handoff/HandoffRequestRegistry.swift`,
   `supacode/Clients/Handoff/HandoffRequestClient.swift`, and
   `supacode/Domain/Handoff/HandoffInjection.swift`.
3. Resolve and freeze Profiles, enforce target exclusivity, split preflight from artifact commit,
   publish typed completion, and add sanitized Profile log metadata in
   `supacode/CLIService/HandoffCommandHandler.swift` and
   `supacode/Domain/Handoff/HandoffCoordinator.swift`.
4. Extend the shared planner and terminal launch boundary in
   `supacode/Domain/AgentProfile/AgentProfileLaunchPlan.swift`,
   `supacode/Clients/Terminal/TerminalClient.swift`,
   `supacode/Features/Terminal/Models/WorktreeTerminalState.swift`, and
   `supacode/Features/Terminal/BusinessLogic/WorktreeTerminalManager.swift`.
5. Build the ordered Profile/runtime target list, preserve execution-time Profile lookup in fallback,
   correlate typed completion, and focus only from a waiting HUD in
   `supacode/Features/App/Reducer/AppFeature+Handoff.swift`,
   `supacode/Features/HandoffHud/Reducer/HandoffHudFeature.swift`, and
   `supacode/Features/HandoffHud/Views/HandoffHudOverlayView.swift`.
6. Wire Profile resolution and the shared launcher once in `supacode/App/supacodeApp.swift`; retain the
   existing Runtime Default launch branch and Profile success-event memory path.
7. Update current behavior in `docs/components/handoff.md`, `docs/components/cli.md`,
   `docs/components/agent-profiles.md`, and `skills/prowl-cli/SKILL.md`; after implementation, record
   actual files, tests, deviations, and PR references in a new 053 action/amendment file.

## Verification

- Parser and socket tests: Profile/runtime forms, both/neither/malformed UUID, explicit Profile source,
  additive payload/text fields, `--no-launch`, and unchanged runtime compatibility.
- Registry/handler tests: source+operation binding, mismatch without consumption, supersession,
  execution-time latest Profile, missing/disabled zero-side-effect rejection, authoritative Profile
  configuration, Runtime Default inheritance, sanitized/no-secret output, and post-commit failure.
- Planner/terminal tests: prompted plan with `-p work`, environment and Dedicated Home intact; handoff
  ignores split placement and creates a background tab; exact result and success/failure events.
- HUD/app tests: Recommended ordering, no-Profile fallback, UUID injection, completion correlation,
  fallback execution-time re-resolution, partial-success messaging, exact-pane focus, and real router
  `--no-launch` wiring.
- Required commands: focused Xcode suites, `swift test --filter HandoffCommandParsingTests`,
  `make build-cli`, `make test-cli-smoke`, `make test-cli-integration`, `make check`, `make test`, and
  `make build-app`.
- Manual GUI pass: a Codex Profile with Extra Arguments `-p work`, environment overrides, and Dedicated
  Home through inline and fallback; deleted/disabled Profile; forced launch failure; Runtime Default
  regression; verify command preview, JSON, logs, and artifacts contain no secret values.

## Alternatives and decisions

- Prowl Agent Profile instead of a Codex-only string keeps handoff runtime-neutral and reuses the full
  launch contract; native Codex profiles remain encapsulated in Extra Arguments.
- UUID instead of display name survives rename and duplicate names.
- Execution-time resolution instead of configuration snapshots avoids credentials in requests and uses
  the latest persisted configuration.
- Reusing the Profile planner/executor instead of copying argv/environment/home logic keeps inline,
  fallback, preview, and ordinary launches consistent.
- Runtime Defaults remain available to avoid a migration cliff and preserve current behavior.

## Open questions

None for this wave.

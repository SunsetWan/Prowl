# Agent Launch and Handoff

This context defines the identities used when Prowl launches a coding agent or transfers a task to a receiving agent.

## Language

**Codex Config Profile**:
A named configuration layer owned by Codex and selected for a Codex launch within the active Codex home.
_Avoid_: Codex profile, Prowl profile

**Prowl Agent Profile**:
A named Prowl-owned launch configuration for one supported agent runtime, optionally representing a distinct account.
_Avoid_: Codex profile, config profile, preset

**Receiving Profile**:
The Prowl Agent Profile selected to launch the agent that takes over a handoff.
_Avoid_: Destination profile, receiving Codex profile

**Runtime Default Target**:
A receiving target that launches a supported agent runtime without selecting a Prowl Agent Profile.
_Avoid_: Bare agent, empty profile

**Receiving Target**:
The launch choice for the agent taking over a handoff: either a Receiving Profile or a Runtime Default Target.
_Avoid_: Destination string, target agent token

**Profile-Aware Handoff**:
A handoff that launches its receiving agent with a selected Prowl Agent Profile instead of only a runtime default.
_Avoid_: Codex-profile handoff, profiled handoff

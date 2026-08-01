# Receiving Profile is authoritative

When a handoff selects a Prowl Agent Profile, that profile exclusively determines the receiver's model, reasoning effort, execution mode, extra arguments, environment, and account; no launch configuration is inherited from the outgoing agent. Existing model and explicitly observed unrestricted-mode inheritance remains only for Runtime Default Targets, preventing an explicit Standard profile from being silently escalated.

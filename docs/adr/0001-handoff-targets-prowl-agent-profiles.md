# Profile-aware handoff targets Prowl Agent Profiles

A profile-aware handoff identifies its receiving configuration with a Prowl Agent Profile rather than accepting runtime-specific input such as a Codex Config Profile name. Existing Runtime Default Targets remain supported for backward compatibility. This keeps the handoff contract runtime-neutral and lets the inline CLI path and HUD fallback launch the same complete profile configuration; native runtime profiles remain encapsulated by the selected Prowl Agent Profile.

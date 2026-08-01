# Handoff profile identity uses UUID

A profile-aware handoff carries the selected Prowl Agent Profile's stable UUID across the HUD, injected CLI request, socket payload, and fallback path; mutable display names are used only for presentation and logs. This prevents a rename or duplicate name from resolving the handoff to a different account or launch configuration.

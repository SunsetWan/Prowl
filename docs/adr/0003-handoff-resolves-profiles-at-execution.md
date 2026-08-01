# Handoff resolves profiles at execution

A profile-aware handoff carries only the Prowl Agent Profile UUID and resolves its latest persisted configuration when the transition executes. A missing or disabled profile fails before any artifact, archive, or log mutation; Prowl does not snapshot profile configuration or place environment values and credentials in the handoff request. The same validation applies to `--no-launch`: a successful archive-only transition records the resolved profile identity but neither creates a receiver surface nor updates `Last Launched Profile`.

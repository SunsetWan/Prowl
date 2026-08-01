# Handoff reuses the Profile launch pipeline

Profile-aware handoff extends and reuses the existing Agent Profile launch planner and surface-launch boundary instead of duplicating profile command, Dedicated Home, environment, and identity handling inside handoff. The shared path accepts a prompted start intent and a handoff-owned background-tab context, while ordinary profile launches retain their current interactive intent and placement behavior.

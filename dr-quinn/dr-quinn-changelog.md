---
project: dr-quinn
file: dr-quinn-changelog
tier: changelog
schema_version: 2
last_updated: 2026-04-15
---

# Dr Quinn — Changelog

Append-only chronological log of meaningful changes to the Dr Quinn state files. Newest entries at the top. One to two lines per entry, stating what changed and why. Loaded only on explicit request — not loaded during normal Q&A or update cycles.

<!-- APPEND_ANCHOR -->

### Update: 15 Apr 2026
v2 architecture migration: monolithic Dr Quinn state file split into live, seven warm files (kiwifruit, carciofi, carciofi-gynae, bunny, fox, protocols, logs), cold archive, and this changelog. Codename map and third-party aliases applied throughout. Geographic identifiers scrubbed for safeguarding. Schema version 2 in effect.

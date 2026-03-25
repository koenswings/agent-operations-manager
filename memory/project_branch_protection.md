---
name: Branch protection policy for agent repos
description: All agent repos on GitHub have protected main branch requiring CEO PR approval, except agent-researcher
type: project
---

All 5 operational agent repos (`koenswings/agent-engine-dev`, `agent-console-dev`, `agent-site-dev`, `agent-quality-manager`, `agent-programme-manager`) have branch protection on `main` — agents cannot push directly, all changes require a PR that the CEO merges.

`agent-researcher` (Compass) does NOT have branch protection — Compass only runs in CEO sessions, no autonomous ops risk.

**Why:** CEO must stay in the loop and approve all agent work products.

**How to apply:** Any architecture for committing files from agent workspaces to GitHub must account for this: identity files and code changes go through PR; memory files need a different strategy (see output 2026-03-21-1200-branch-protection-and-memory-files.md).

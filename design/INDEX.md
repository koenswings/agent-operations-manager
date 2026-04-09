# Design Document Index — Atlas (agent-operations-manager)

Agent-local design working files. See `../../design/INDEX.md` for org-level design docs.

Update this file whenever a local design doc is added, superseded, or its status changes.

---

## mc-deeper-integration.md
**Status:** Draft  ·  **Date:** 2026-04-09  ·  **Author:** Atlas
Exploration of routes for shifting more of the CEO workflow into Mission Control. Full MC UI
feature reference with IDEA-specific assessments. Three integration routes: Route A (task
comments as async channel), Route B (dual surface with Telegram commands + GitHub webhooks),
Route C (MC primary). Recommendation: start with Route A.
→ [design/mc-deeper-integration.md](mc-deeper-integration.md)

## openclaw-initial-config/
**Status:** Archived  ·  **Date:** 2026-03-25
Working notes from the initial OpenClaw configuration and agent setup process. No longer
actively referenced — superseded by the implemented platform config in `platform/`.

## tmux-vscode-setup/
**Status:** Archived  ·  **Date:** 2026-03-25
Draft notes for tmux/VS Code dev environment setup. Superseded — tmux dev environment
was removed from IDEA's tooling (see agent-engine-dev PR #14).

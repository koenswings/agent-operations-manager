# Design Document Index — Atlas (agent-operations-manager)

Agent-local design working files. See `../../design/INDEX.md` for org-level design docs.

Update this file whenever a local design doc is added, superseded, or its status changes.

---

## mc-deeper-integration.md
**Status:** Archived  ·  **Date:** 2026-04-09  ·  **Decided:** 2026-04-11  ·  **Author:** Atlas
Exploration of routes for shifting more of the CEO workflow into Mission Control. After review,
decision was made to proceed with Telegram as the primary interface. MC remains a task register
only (boards, task status, agent health). No further MC integration routes will be pursued.
→ [design/mc-deeper-integration.md](mc-deeper-integration.md)

## openclaw-initial-config/
**Status:** Archived  ·  **Date:** 2026-03-25
Working notes from the initial OpenClaw configuration and agent setup process. No longer
actively referenced — superseded by the implemented platform config in `platform/`.

## tmux-vscode-setup/
**Status:** Archived  ·  **Date:** 2026-03-25
Draft notes for tmux/VS Code dev environment setup. Superseded — tmux dev environment
was removed from IDEA's tooling (see agent-engine-dev PR #14).

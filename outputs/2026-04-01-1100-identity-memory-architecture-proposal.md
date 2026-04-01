> **Task/Question:** Write a design proposal for the agent identity and memory architecture.

## Proposal

idea PR [#21](https://github.com/koenswings/idea/pull/21) — `proposals/agent-identity-memory-architecture`

Full proposal at: `proposals/2026-04-01-agent-identity-memory-architecture.md`

## Summary

Three-layer architecture replacing the memory/updates branch workflow:

**Layer 1 — Identity:** `idea/identity/[agent]/` on main. All Group 1 files (AGENTS.md,
SOUL.md, IDENTITY.md, USER.md, TOOLS.md, HEARTBEAT.md, CLAUDE.md) per agent. Branch-protected,
PR-reviewed, Atlas maintains. Changes via: Atlas edits → Telegram notify → PR merge.

**Layer 2 — Memory:** `idea-memory` repo (new, no branch protection). All Group 3 files
(memory/YYYY-MM-DD.md, MEMORY.md, outputs/). Cron-backed hourly — agents write locally,
never push. Two repos to clone on restore.

**Layer 3 — Working code:** Agent repos unchanged.

## Key decisions captured in proposal

- Memory vs outputs distinction: memory = summarised traces (read at boot); outputs = full
  traces (never auto-loaded, audit only)
- Cron is the only backup mechanism — agents never run git for memory
- Identity change protocol: Atlas edits file → Telegram notify via Koen → agent aware now →
  PR for persistence
- Install/restore: two clones + two copy steps
- 5 open questions flagged (OpenClaw path config, idea-memory visibility, cron frequency,
  MEMORY.md scope, outputs retention)

## Open memory/updates PRs — status

These PRs contain the AGENTS.md updates (git fetch, cross-agent comms). Still need merging
before this proposal can be executed (they update AGENTS.md which will eventually move to
idea/identity/).

- Atlas [#10](https://github.com/koenswings/agent-operations-manager/pull/10)
- Axle [#27](https://github.com/koenswings/agent-engine-dev/pull/27)
- Pixel [#17](https://github.com/koenswings/agent-console-dev/pull/17)
- Beacon [#12](https://github.com/koenswings/agent-site-dev/pull/12)
- Marco [#19](https://github.com/koenswings/agent-programme-manager/pull/19)

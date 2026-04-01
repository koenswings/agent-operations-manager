> **Task/Question:** Finalise the identity & memory architecture design with flowcharts. Repo name: agent-identities. Delete memory/updates branches when clean.

## Summary

Wrote and opened PR for `design/agent-identity-memory-architecture.md`.

**PR:** https://github.com/koenswings/idea/pull/22  
**Branch:** `design/agent-identity-memory-architecture`

## Design decisions captured

- **One backup repo**: `agent-identities` — identity + memory files for all 5 agents, subdirs per agent
- **Not branch-protected** — Atlas pushes directly, no PR ceremony
- **Nightly Pi cron at 03:00 UTC** — rsync workspace → repo; diff identity files; alert Atlas on drift
- **Drift detection**: identity file change → Telegram alert to Atlas with file+line summary → Atlas ratifies (no action) or reverts (`git show HEAD:...` → write back to workspace)
- **MEMORY.md for all agents** — boot sequence step added; agents write freely; no drift alert for MEMORY.md
- **Agent code repos gitignore** both identity and memory file groups — pure code repos
- **Replaces** memory/updates branch + PR flow entirely

## Four Mermaid flowcharts in the doc

1. Authorised identity change flow
2. Drift detection + Atlas review flow  
3. Memory file lifecycle (including re-hosting recovery)
4. Nightly backup cron

## Memory/updates branches — NOT deleted yet

All 5 branches have pending ops: commits (AGENTS.md changes for ff-only fix, cross-agent comms, PR link rule). Cannot safely delete until those are cherry-picked into proper PRs and merged. Listed as step 7 in migration section of design doc.

## Next steps (post-approval)

1. Koen creates `agent-identities` repo on GitHub (koenswings/agent-identities)
2. Atlas seeds repo with current files from all 5 workspaces
3. Atlas adds backup script to idea/scripts/ + cron entry
4. Atlas adds .gitignore to all 5 agent repos
5. Atlas adds MEMORY.md step + starter files for Axle, Pixel, Beacon, Marco
6. Atlas cherry-picks ops: commits → clean PR per agent → delete memory/updates
7. Atlas updates PROCESS.md + all AGENTS.md to remove memory/updates workflow

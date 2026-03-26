> **Task/Question:** Is Atlas well-known to the other agents? Check please. And enable a board for him by repurposing Veri's board.

## Atlas visibility audit

| Agent | Knew Atlas reviews? | Had cross-agent instructions? |
|-------|--------------------|-----------------------------|
| Axle (engine-dev) | ✅ "Atlas (operations-manager) reviews; CEO merges" | ❌ None |
| Pixel (console-dev) | ✅ "Atlas (operations-manager) reviews; CEO merges" | ❌ None |
| Beacon (site-dev) | ❌ Atlas not mentioned at all | ❌ None |
| Marco (programme-manager) | ✅ "Atlas reviews external-facing drafts" | ❌ None |

**Gaps found:** Beacon had no Atlas mention; no agent knew how to create a cross-agent request.

## Actions taken

### Script: Atlas board enabled
- `scripts/check-new-tasks.sh` — Atlas entry added using Veri's repurposed board:
  - Board: `d0cfa49e-edcb-4a23-832b-c2ae2c99bf67`
  - Agent ID: `operations-manager`

### Beacon AGENTS.md — Atlas added to Development Workflow
- Step 6 now reads: "Atlas (operations-manager) reviews; CEO merges → GitHub Actions deploys"

### All four agents — Cross-Agent Requests section added
Each agent now has a `## Cross-Agent Requests` section with:
- How to create a request (tag: `cross-agent`, typed title prefix)
- A table of relevant boards with "when to use" guidance, tailored per agent:

| Agent | Atlas board | Axle board | Marco board |
|-------|------------|-----------|------------|
| Axle | Reviews, architecture | — | Marco (field viability) |
| Pixel | Reviews, consistency | Engine API questions | — |
| Beacon | Reviews | — | Content clarification |
| Marco | External draft review | Technical feasibility | — |

## PRs

- idea PR #5 updated (`docs/cross-agent-requests`) — script change added
- engine-dev PR #13 updated (memory/updates)
- console-dev PR #9 updated (memory/updates)
- site-dev PR #9 updated (memory/updates)
- programme-manager PR #16 opened (memory/updates)

> **Question:** How is it possible that no remote is configured for the agents — each agent has a repo on GitHub (koenswings/agent-engine-dev etc.)

## Two Directories Per Agent

Each agent has **two separate directories** on the Pi. They are not linked to each other.

### The two locations

| Directory | Purpose | Remote | Latest state |
|-----------|---------|--------|-------------|
| `/home/pi/idea/agents/agent-{role}/` | The GitHub repo — code + identity files | ✓ `koenswings/agent-{role}` | Older — missing operational flow additions |
| `/home/pi/idea/workspace-lead-{board_id}-*/` | OpenClaw workspace — what the running agent actually reads | ✗ none | Up to date — includes all audit changes |

### What happened

OpenClaw/Mission Control provisions agents by creating a workspace directory and initialising it as a local git repo. It does **not** clone the GitHub repo into that directory — the workspace is a separate local repo with no remote wired up.

All the identity file edits we've made (SOUL.md, AGENTS.md, HEARTBEAT.md, MEMORY.md) were applied to the **workspace dirs** — because that's what the running agents read. The **GitHub repos** in `/home/pi/idea/agents/agent-*/` have not been updated and are behind.

### Evidence — SOUL.md line count

| Agent | GitHub repo SOUL.md | Workspace SOUL.md |
|-------|--------------------:|------------------:|
| Axle (engine-dev) | 49 lines | 68 lines |
| Pixel (console-dev) | 49 lines | 68 lines |
| Beacon (site-dev) | 49 lines | 68 lines |
| Veri (quality-manager) | 49 lines | 68 lines |
| Marco (programme-manager) | 49 lines | 68 lines |

The 19 extra lines in each workspace SOUL.md = the `## How Work Flows` section added during the operational flow audit.

### What needs to happen

The GitHub repos need to be brought up to date with the workspace files. For each agent:
- `SOUL.md` — missing `## How Work Flows` (operational model, auto-review hard rule, heartbeat definition)
- `AGENTS.md` — missing `## Your Role` (role-specific scope) and `## Cross-Agent Review Protocol`
- `HEARTBEAT.md` — missing `## External Event Monitoring (Future Heartbeats)` section
- `MEMORY.md` — missing Telegram group name and chat ID

The simplest fix: copy the updated files from the workspace dirs into the agent repos, commit, and push.

### Which is authoritative going forward

The **workspace dir** is what the running agent reads — it is the operational truth. The **GitHub repo** should be the backup and source of record. They should be kept in sync.

One option: when making identity file changes, make them in the GitHub repo first (edit → commit → push), then copy into the workspace dir. This makes GitHub the canonical source and keeps the history clean. The reverse (workspace → GitHub) works too but is harder to automate.

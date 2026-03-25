> **Question:** Please clean up the workspace files and move them into an /home/pi/obsolete folder. Also move everything outside the idea folder that is not relevant anymore into this obsolete folder. Why are you saying 'Once merged, agents will find their identity files and memory at the code repo paths' if agents already have their memory now in the local copy of the repo from which the PR was pushed?

---

## Correction: "Once merged" was misleading

All five operational agent repos are on the `setup/workspace-migration` branch locally. That branch already contains `memory/`, `outputs/.gitkeep`, and the updated identity files. Since agents read from the local filesystem — not from GitHub — they can access all of these files right now, without waiting for the PR to merge.

The PR merge is only required for GitHub's `main` branch to reflect the current state. It exists for governance and audit trail. It does not gate the agents' ability to use the files.

The "Once merged" phrasing was wrong. The files are live.

---

## Cleanup executed

### Moved to `/home/pi/obsolete/`

| Item | Source | Reason |
|------|--------|--------|
| `workspace-lead-6bddb9d2-*` | `/home/pi/idea/` | Board lead workspace dir — replaced by `agents/agent-engine-dev/` |
| `workspace-lead-ac508766-*` | `/home/pi/idea/` | Board lead workspace dir — replaced by `agents/agent-console-dev/` |
| `workspace-lead-7cc2a1cf-*` | `/home/pi/idea/` | Board lead workspace dir — replaced by `agents/agent-site-dev/` |
| `workspace-lead-d0cfa49e-*` | `/home/pi/idea/` | Board lead workspace dir — replaced by `agents/agent-quality-manager/` |
| `workspace-lead-3f1be9c8-*` | `/home/pi/idea/` | Board lead workspace dir — replaced by `agents/agent-programme-manager/` |
| `workspace-gateway-5632ed12-*` | `/home/pi/idea/` | Empty gateway agent workspace — never configured |
| `workspace-gateway-ccc8463b-*` | `/home/pi/idea/` | Old gateway agent workspace (OpenClaw Pi Gateway Agent) — not in current config |
| `projects/` | `/home/pi/` | Pre-configured agent workspaces from February (engine, backups, idea-proposal) — entirely superseded by current setup |
| `setup-github.sh` | `/home/pi/` | One-time GitHub repo setup script — repos already created and configured |

### Kept

| Item | Location | Reason |
|------|----------|--------|
| `idea/` | `/home/pi/` | Org root — all active work |
| `openclaw/` | `/home/pi/` | Docker compose setup for OpenClaw and Mission Control — still needed |
| `boot.out` | `/home/pi/` | System boot log — not workspace content |

### `/home/pi/idea/` — final state

```
agents/
  agent-researcher/     ← Compass (this agent)
  agent-engine-dev/     ← Axle
  agent-console-dev/    ← Pixel
  agent-site-dev/       ← Beacon
  agent-quality-manager/ ← Veri
  agent-programme-manager/ ← Marco
engine/                 ← shared engine project (not an agent workspace)
BACKLOG.md
CONTEXT.md
PROCESS.md
ROLES.md
logs/
scripts/
standups/
```

The `workspace-lead-*` and `workspace-gateway-*` dirs are gone. The `idea/` root is clean.

---

## Note on `memory/` in MEMORY.md

The MEMORY.md entry for `md-to-pdf` referenced `/home/pi/projects/engine/md-to-pdf`. That path no longer exists — `projects/` has been moved to `obsolete/`. The md-to-pdf capability needs to be located elsewhere before next use (check `engine/` in `/home/pi/idea/`).

---

*Compass · Strategic Advisor · IDEA*

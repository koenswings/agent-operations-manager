> **Question:** Where are all non-PR output files stored, in which repo, are they properly checked in, and what actions are still needed?

## Complete Output Retention Overview

---

### 1. Compass (agent-researcher) — research outputs and session outputs

**Location:** `/home/pi/idea/agents/agent-researcher/`
**GitHub repo:** `koenswings/agent-researcher` (unprotected, direct push to main)

| Path | What it contains | Committed? | Pushed to GitHub? |
|------|-----------------|------------|-------------------|
| `outputs/*.md` | All session outputs (answers to CEO questions) | ✓ Yes | ✗ 3 commits unpushed |
| `research/openclaw-initial-config/` | Living design documents, staged agent files | ✓ Mostly | ✗ 3 commits unpushed |
| `research/openclaw-initial-config/agent-engine-dev/SOUL.md` | Modified this session | ✗ Not staged | — |
| `memory/` | Compass's own memory files | ✓ Yes | ✗ 3 commits unpushed |

**Status:** 3 commits locally committed but not pushed to GitHub. 1 file modified and not yet committed.

---

### 2. Operational agent workspace dirs — identity files and memory

Each agent has a workspace dir: `/home/pi/idea/workspace-lead-{board_id}-*/`
These are **local git repos only — no remote, no GitHub backup.**

#### Axle — engine-dev (`workspace-lead-6bddb9d2-...`)

| File | Status |
|------|--------|
| SOUL.md | Modified, uncommitted |
| AGENTS.md | Modified, uncommitted |
| HEARTBEAT.md | Modified, uncommitted |
| MEMORY.md | Modified, uncommitted |
| memory/2026-03-01.md | Committed (in the one init commit) |
| memory/2026-03-19.md | Committed (in the one init commit) |

1 commit total. Changes from the 2026-03-21 audit session are uncommitted.

#### Pixel — console-dev (`workspace-lead-ac508766-...`)

| File | Status |
|------|--------|
| SOUL.md | Modified, uncommitted |
| AGENTS.md | Modified, uncommitted |
| HEARTBEAT.md | Modified, uncommitted |
| MEMORY.md | Modified, uncommitted |
| memory/2026-03-01.md | Committed |
| memory/2026-03-02.md | Committed |
| memory/2026-03-19.md | Committed |
| memory/2026-03-20.md | Committed |

1 commit total. Same pattern as Axle.

#### Veri — quality-manager (`workspace-lead-d0cfa49e-...`)

| File | Status |
|------|--------|
| SOUL.md | Modified, uncommitted |
| AGENTS.md | Modified, uncommitted |
| HEARTBEAT.md | Modified, uncommitted |
| MEMORY.md | Modified, uncommitted |
| TOOLS.md | Modified, uncommitted |
| BOOTSTRAP.md | Untracked (never committed) |
| memory/2026-03-01.md | Committed |
| memory/2026-03-03.md | Untracked (never committed) |
| memory/2026-03-19.md | Untracked (never committed) |

1 commit total. Also has untracked memory files and an untracked BOOTSTRAP.md.

#### Marco — programme-manager (`workspace-lead-3f1be9c8-...`)

**Everything untracked — no commits at all.**

| File | Status |
|------|--------|
| SOUL.md | Untracked |
| AGENTS.md | Untracked |
| HEARTBEAT.md | Untracked |
| IDENTITY.md | Untracked |
| MEMORY.md | Untracked |
| TOOLS.md | Untracked |
| USER.md | Untracked |
| memory/ | Untracked |
| api/ | Untracked |

#### Beacon — site-dev (`workspace-lead-7cc2a1cf-...`)

**Everything untracked — no commits at all.**

Same pattern as Marco. No commits.

---

### 3. Operational agent project repos — identity files (not workspace)

**Location:** `/home/pi/idea/agents/agent-{role}/`
**GitHub repos:** Protected, CEO merges only.

These are behind the workspace dirs — they have the old (pre-audit) versions of identity files. The workspace dirs have the current state; the project repos have not been updated. There is no active retention concern here (they're committed and pushed) but they are stale.

---

### Summary: retention health

| Storage location | GitHub backup? | Committed? | Risk |
|-----------------|----------------|------------|------|
| Compass outputs + research | ✓ Yes (minus 3 unpushed commits) | ✓ Yes | Low — Pi loss would lose 3 commits |
| Workspace identity files (SOUL, AGENTS, etc.) | ✗ No | ✗ Mostly not | **High — Pi loss loses all audit changes** |
| Workspace memory files | ✗ No | Partial | **High — Pi loss loses all session memory** |
| Project repo identity files | ✓ Yes | ✓ Yes (stale) | Low — safe but out of date |

The biggest exposure: all the work done in the 2026-03-21 audit session (SOUL.md, AGENTS.md, HEARTBEAT.md updates for all 5 agents) exists only in workspace dirs, uncommitted, with no remote. It lives entirely on the Pi's filesystem.

---

### Actions to take

#### Immediate (Compass session — today)

1. **Commit the modified SOUL.md** (`research/openclaw-initial-config/agent-engine-dev/SOUL.md`)
2. **Push agent-researcher to GitHub** — 3 commits + the SOUL.md commit are unpushed

#### Workspace dirs — commit the audit changes

For each of the 5 workspace dirs, commit the outstanding changes. These are all owned by root so require `sudo`.

```bash
# Example for Axle — repeat for each workspace dir
cd /home/pi/idea/workspace-lead-6bddb9d2-c06f-444d-8b18-b517aeaa6aa8
sudo git add SOUL.md AGENTS.md HEARTBEAT.md MEMORY.md
sudo git commit -m "Update identity files from 2026-03-21 audit"
```

For Marco and Beacon (no commits at all), add everything:
```bash
sudo git add .
sudo git commit -m "Initial commit: workspace files + 2026-03-21 audit state"
```

For Veri, also add the untracked files:
```bash
sudo git add BOOTSTRAP.md memory/2026-03-03.md memory/2026-03-19.md
```

#### Retention model gap — no remote for workspace dirs

Even after committing, workspace dirs have no remote. If the Pi is lost or the filesystem corrupted, all workspace memory is gone.

**Decision needed:** How to back up workspace dirs. Two options:

**Option A — No backup (accept the risk)**
Workspace memory is semi-ephemeral. Agents can re-bootstrap from identity files in the project repos. Memory loss means agents lose session context but not their core identity. If this is acceptable, no action needed beyond local commits.

**Option B — Backup remote (low overhead)**
Add a remote to each workspace dir pointing to a GitHub repo (could be the same project repo on a separate branch, or a dedicated `memory` repo). A nightly cron pushes. Adds resilience without touching the PR model.

**Current recommendation:** Option A for now — the workspace dirs contain operational state, not irreplaceable data. The canonical identity files live in the project repos and in Compass's research files. Agents can be re-provisioned. This is a known acceptable risk, not an oversight.

#### Project repos — sync stale identity files

The project repos still have pre-audit identity files. To sync:
1. Compass stages the updated files from `research/openclaw-initial-config/agent-*/`
2. CEO opens and merges a PR per repo (or per batch if GitHub allows)
3. Workspace dirs already have the correct files — no further sync needed there

This is a governance task, not urgent for daily operations.

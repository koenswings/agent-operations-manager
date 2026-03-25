> **Question:** Where are all the non-PR output files stored, in which repo, and are they already properly checked-in?

## Workspace Git State — Audit

### Short answer

Non-PR outputs (design docs, proposals, reports) from operational agents would live in their **workspace directories** (`/home/pi/idea/workspace-lead-{board_id}-*/`). These directories are local git repos — but **no remote is configured** on any of them, and recent changes are **uncommitted**. Nothing is pushed anywhere.

---

### Two separate storage locations

| What | Where | Repo | Remote | Status |
|------|-------|------|--------|--------|
| Compass research & outputs | `/home/pi/idea/agents/agent-researcher/` | `koenswings/agent-researcher` | ✓ configured | ✓ committed and pushed |
| Agent workspace files (SOUL, AGENTS, MEMORY etc.) | `/home/pi/idea/workspace-lead-{board_id}-*/` | local git only | ✗ no remote | ✗ changes uncommitted |

---

### Workspace git state (all 5)

| Agent | Board ID | Uncommitted changes | Untracked files | Remote |
|-------|----------|--------------------:|----------------:|--------|
| Axle | `6bddb9d2` | AGENTS.md, HEARTBEAT.md, MEMORY.md, SOUL.md | .claude/ | none |
| Pixel | `ac508766` | AGENTS.md, HEARTBEAT.md, MEMORY.md, SOUL.md | .claude/ | none |
| Beacon | `7cc2a1cf` | — | everything (never committed) | none |
| Veri | `d0cfa49e` | AGENTS.md, HEARTBEAT.md, MEMORY.md, SOUL.md, TOOLS.md, memory/2026-03-01.md | .claude/, BOOTSTRAP.md, api/, 2 memory files | none |
| Marco | `3f1be9c8` | — | everything (never committed) | none |

Beacon and Marco workspaces have **never had a commit** — all files are untracked.

---

### What this means

1. **Compass outputs are safe** — `agent-researcher` repo is on GitHub, up to date.

2. **Agent workspace files are not backed up** — if the Docker volume is lost or a workspace is reprovisioned, the identity/memory files are gone. The canonical versions are in `research/openclaw-initial-config/agent-*/` in the researcher repo, but those don't include post-deployment changes (SOUL.md additions, MEMORY.md Telegram entries etc.).

3. **No remote = no history** — agent Claude memory logs (`memory/2026-03-*.md`) exist only on the Pi.

---

### Options

**Option A — Add remotes, commit and push workspace files**
- Create one repo per agent in the GitHub org (`idea-edu-africa/agent-engine-dev` etc.) or use one shared `idea` repo
- Wire up remotes, commit current state, push
- Future changes (memory log entries, identity file edits) would need to be committed/pushed — either manually or via a post-session hook

**Option B — Treat the research folder as canonical, sync back periodically**
- Keep `research/openclaw-initial-config/agent-*/` up to date with any changes made to workspace files
- Workspaces are ephemeral; research repo is the source of truth
- Simpler, but memory logs (`memory/*.md`) still aren't backed up

**Option C — Do nothing for now**
- Workspace files are stable after the audit; risk is low unless the Pi is rebuilt
- Defer until Step 16/17 (app-openclaw repo and idea repo setup), which would naturally include a backup strategy

---

### Recommendation

The cleanest long-term solution is **Option A** as part of Steps 16–17 — the `idea` repo (`idea-edu-africa/idea`) could include agent workspace files, or each agent gets its own repo in the org. But this is a build task. For now, Option C is safe unless you're concerned about Pi data loss.

> **Question:** Will this be future-proof? What about the output that each agent produces and any other updates? Will these be committed automatically?

## Output Storage Architecture — Gap Analysis

### Short answer

**No, not future-proof as-is.** Code PRs work correctly. Everything else (design docs, proposals, reports, memory updates) has no reliable commit/push path. Three separate gaps.

---

### What works today

**Code PRs** — Agents know their project repo lives at `/home/node/workspace/agents/agent-{role}/` (confirmed in Axle's MEMORY.md: `Engine repo at /home/node/workspace/agents/agent-engine-dev/`). This maps to `/home/pi/idea/agents/agent-{role}/` on the Pi — which IS the GitHub-connected repo with a remote. When an agent:
1. Creates a feature branch
2. Commits changes
3. Pushes and opens a PR

…it goes to GitHub. This path works end-to-end.

---

### Gap 1 — `/home/pi/idea/` is not a git repo

SOUL.md instructs agents to write non-code outputs relative to `../../` from their workspace path:

| Output type | Destination (from workspace dir) | Absolute path |
|-------------|----------------------------------|---------------|
| Design doc | `../../design/` | `/home/pi/idea/design/` |
| Proposal | `../../proposals/` | `/home/pi/idea/proposals/` |
| Report | committed directly (no PR) | unclear |

`/home/pi/idea/` has no `.git` directory. Design docs and proposals written there exist only on the Pi's filesystem — **not in any git repo**.

**Fix:** The `idea` repo (`idea-edu-africa/idea`) was planned in Steps 16–17. This repo needs to be cloned to `/home/pi/idea/` (or the org root files need to live there) so that `design/`, `proposals/`, and the org-root files (`CONTEXT.md`, `ROLES.md` etc.) are tracked. Without this, every design doc and proposal the agents produce is untracked.

---

### Gap 2 — Workspace dirs have no remote

The workspace dirs (`/home/pi/idea/workspace-lead-{board_id}-*/`) are what agents actually read during a session. They contain:
- Identity files: SOUL.md, AGENTS.md, MEMORY.md etc.
- Session memory logs: `memory/YYYY-MM-DD.md`
- API spec: `api/openapi.json`, `api/agent-lead-operations.tsv`

None of these are pushed anywhere. Memory logs (which accumulate agent thinking/context between sessions) exist only locally.

**Fix options:**
- **Option A:** Wire workspace dirs to the GitHub agent repos as remotes (e.g. `git remote add origin git@github.com:koenswings/agent-engine-dev.git`). Agents could then commit and push identity + memory changes directly. The workspace only has identity files, not full project code — pushing would add only those files to the repo. This is feasible but means the agent repo holds both project code and identity/memory files.
- **Option B:** Keep workspace dirs local-only; add a nightly cron that copies changed workspace files into the agent repos, commits, and pushes. Simpler to reason about but has a lag.
- **Option C:** Sparse-checkout the agent repo into the workspace dir (identity subdirectory only). Most architecturally clean but requires restructuring the agent repos.

---

### Gap 3 — No auto-commit/push mechanism

Even if the remotes were wired up, there is nothing that automatically commits and pushes after a session ends. An agent can write a file but won't necessarily commit it — unless it's instructed to do so.

Currently SOUL.md says to commit outputs, but:
- There is no post-session hook that forces a commit/push
- Agents running via OpenClaw/MC don't have a guaranteed "session end" trigger
- If the agent crashes or is killed mid-session, uncommitted work is lost

**Fix options:**
- **Claude Code hook** — add a `Stop` hook in the workspace `.claude/settings.json` that runs `git add -A && git commit -m "session: $(date)" && git push` automatically when a session ends. This is the most reliable mechanism.
- **Cron-based commit** — a Pi cron that runs every few minutes, finds uncommitted changes in workspace dirs, and commits/pushes them. Simpler but adds commit noise and doesn't catch mid-session crashes.

---

### Recommended path (in priority order)

| Priority | Fix | What it unblocks |
|----------|-----|-----------------|
| 1 | Clone/init `idea` repo at `/home/pi/idea/` | Design docs and proposals land in git (Steps 16–17) |
| 2 | Wire workspace dir remotes to GitHub agent repos | Identity and memory files backed up |
| 3 | Add `Stop` hook in workspace `.claude/settings.json` | Auto-commit/push on session end |

Priority 1 is the same work as Steps 16–17 (create the `idea` repo and configure it). Priorities 2 and 3 are small additions that can be done immediately or alongside 1.

---

### Summary table

| Output type | Current status | Fix needed |
|-------------|---------------|------------|
| Code PR | ✓ Works — agent repos have remotes | None |
| Design doc | ✗ Written to untracked `/home/pi/idea/design/` | Gap 1: `idea` repo |
| Proposal | ✗ Written to untracked `/home/pi/idea/proposals/` | Gap 1: `idea` repo |
| Report | ✗ Unclear commit target | Gap 1: `idea` repo |
| Identity files (SOUL, AGENTS etc.) | ✗ Workspace-only, no remote | Gap 2: wire remotes |
| Memory logs (`memory/*.md`) | ✗ Workspace-only, no remote | Gap 2: wire remotes |
| Auto-commit on session end | ✗ No mechanism | Gap 3: Stop hook or cron |

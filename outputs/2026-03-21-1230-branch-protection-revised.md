> **Question:** Does the fact that agent-researcher is interacted with via Claude (Code), while other agents interact via Telegram through OpenClaw, change the branch protection / memory files analysis?

## Branch Protection and Memory Files — Revised Analysis

### The new context

The interaction model is asymmetric:

| Agent | Interface | Capability |
|-------|-----------|-----------|
| Compass (researcher) | Claude Code (CLI) | Full filesystem access, git, any shell command |
| Operational agents (all 5) | Telegram → OpenClaw | Sandboxed; only tools defined in TOOLS.md |

This matters a lot. The original analysis treated all agents as roughly equivalent actors. They are not.

---

### What changes

#### 1. Memory file concern is already resolved for operational agents

The previous analysis worried that branch protection would block memory commits because agents need to commit memory immediately, before the CEO reviews anything.

That concern doesn't apply to operational agents via Telegram/OpenClaw, because their memory files are in **workspace dirs** (`/home/pi/idea/workspace-lead-*/`), which already have no remote and are not subject to GitHub branch protection at all. They commit locally; nothing is gated.

The concern was real for a hypothetical where memory and code co-exist in a single protected repo. But the two-directory architecture (workspace dir = operational state, project repo = canonical identity/code) already separates them. The worry is solved by structure, not by adding exceptions.

**Verdict:** Memory files are fine. No design change needed.

#### 2. The "who creates PRs" question is now answered clearly

The previous analysis flagged this as an open design question:

> *"This raises a design question about who owns identity file updates: if only the CEO can merge to protected repos, and operational agents can't create PRs for their own identity files..."*

The Telegram/OpenClaw interface answers this directly: **operational agents cannot create GitHub PRs at all.** They operate within OpenClaw's sandbox. Their tools (TOOLS.md) are infrastructure endpoints, not GitHub API calls. There is no PR capability in a Telegram chat session.

This means:
- Operational agents cannot self-modify their own identity files in the project repos — even if we wanted them to.
- Compass is the *only* agent that can practically touch GitHub, because Compass runs in Claude Code with full shell access.

This makes Compass-as-architectural-authority not just preferred, but structurally enforced by the interface difference.

#### 3. Branch protection on operational repos is CEO ↔ Compass governance, not CEO ↔ agent governance

The original framing was: branch protection keeps the CEO in the loop on what agents do to their own repos.

The corrected framing: operational agents never interact with their project repos at all. The only actors touching those repos are the CEO and Compass. Branch protection on operational repos is therefore a **Compass → CEO approval gate**, not an agent self-modification guard.

This reframes the risk model:
- Old risk: agents could rewrite SOUL.md without CEO oversight.
- Actual risk: Compass could update identity files and push them to project repos without CEO review — since Compass has full git access and agent-researcher is unprotected.

The branch protection on operational repos correctly blocks this. Compass cannot push directly to `agent-engine-dev` (protected). Changes must go through a PR. CEO reviews and merges. This is the right design, now understood from the right angle.

#### 4. Compass is the architectural chokepoint — in a good way

The two-tier model means:

```
CEO
 ↕  (Claude Code — full access)
Compass
 ↕  (PR, CEO merges)
Operational project repos (protected)
         ↓  (sync, manual or scripted)
Workspace dirs (OpenClaw-managed, no remote)
         ↕  (Telegram)
Operational agents
         ↕  (Telegram)
CEO
```

Compass is the only actor that can move identity changes from design into the canonical repos. The CEO is the only actor that can approve those changes via PR merge. Operational agents see the result in their workspace dirs but cannot alter the source.

This is clean. No agent can self-modify its own governance documents. The architectural authority is explicit and interface-enforced, not just policy-enforced.

---

### What doesn't change

- **Identity files (SOUL, AGENTS, HEARTBEAT, BOOTSTRAP, IDENTITY): CEO review required.** Unchanged. Branch protection is correct.
- **Memory files: no CEO review.** Still correct, and now additionally confirmed by the workspace dir separation.
- **Compass as identity file owner:** Confirmed and now structurally enforced.
- **The sync gap:** Still exists. Project repos are behind workspace dirs. The fix direction remains: Compass stages changes in `research/openclaw-initial-config/agent-*/`, opens a PR for CEO review, CEO merges to project repo, then workspace dirs are manually synced.

---

### One new implication: Compass needs a PR-creation capability

Since Compass is the only practical author of identity file PRs, and Compass currently operates against its own unprotected repo, the workflow requires Compass to:

1. Maintain canonical identity files in `research/openclaw-initial-config/`
2. When an update is ready, push a branch to the relevant protected repo and open a PR
3. CEO reviews and merges

For this to work, Compass needs either:
- A GitHub Personal Access Token in its Claude Code environment, or
- The CEO to run the git push / PR creation step themselves after Compass prepares the files

Currently neither is explicitly set up. The CEO has been making git changes directly. This is fine for now but worth noting as the organisation matures.

---

### Revised summary

| Question | Previous answer | Revised answer |
|----------|----------------|----------------|
| Do memory files need PR protection? | No — immediacy requirement | No — and additionally: workspace dirs already have no remote |
| Who can create PRs for identity files? | Open question | Compass only (structural, not just policy) |
| What does branch protection on operational repos guard against? | Agents self-modifying | Compass pushing without CEO review |
| Is the design correct? | Yes, with memory file exclusion | Yes — and the interface asymmetry reinforces it cleanly |

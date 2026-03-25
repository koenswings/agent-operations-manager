> **Question:** Branch protection is on all agent repos except agent-researcher to put the CEO in the loop. But I'm not sure memory and identity files need to be reviewed. Please advise.

## Branch Protection and Memory Files — Governance Design

### The core question

Branch protection exists to keep the CEO in the loop on changes that affect agent behaviour or work products. The question is whether ALL files warrant that review, or whether some can bypass it.

The answer: **it depends on what the file controls.** There are three distinct categories, not two.

---

### Three categories of files

#### Category 1 — Behaviour files (SOUL.md, AGENTS.md, HEARTBEAT.md, BOOTSTRAP.md, IDENTITY.md)

These define what an agent IS and how it operates. SOUL.md contains the hard rules (auto-review guard, no autonomous work). AGENTS.md defines role scope and the cross-agent protocol. Changing these changes agent behaviour.

**Verdict: PR required. CEO reviews.**

Rationale: If an agent could rewrite its own SOUL.md without CEO oversight, it could remove safety guards. These are governance documents, not operational notes. Changes are rare and intentional — always directed by Compass or the CEO. The PR overhead is trivial relative to the risk.

#### Category 2 — Configuration files (TOOLS.md, USER.md)

TOOLS.md contains infrastructure credentials (BASE_URL, AUTH_TOKEN, API endpoints). USER.md is the CEO's own profile — preferences, communication style, things the agent should know about working with you. Neither defines agent behaviour in the SOUL.md sense.

**Verdict: PR not required for USER.md (CEO's own profile). PR can be waived for TOOLS.md changes too** — these are infrastructure, not governance. In practice these rarely change and could go through PR for the audit trail.

#### Category 3 — Memory files (MEMORY.md, `memory/YYYY-MM-DD.md` session logs)

This is where your instinct is right that something is different. Memory files are an agent's running operational state — what it's working on, what changed in the last session, what to pick up next. Session logs (`memory/*.md`) are essentially a diary.

**Verdict: CEO review NOT needed. Direct commit.**

Reasons:
1. **Immediacy requirement.** Memory needs to be committed and available at the START of the next session. If a memory update requires a PR that the CEO reviews and merges, the agent starts the next session with stale or missing context. The system breaks.
2. **The auto-review workflow depends on it.** When Veri reviews Axle's PR and writes a response, that session needs to commit its memory (what it reviewed, what it concluded) immediately — before the CEO has seen anything. A PR gate here creates a deadlock.
3. **You don't review employees' personal notes.** Memory is what the agent knows about its own state, not a work product. You evaluate agents by what they produce (PRs, design docs, proposals), not by what they write in their own notebooks.
4. **Branch protection can't accommodate this gracefully.** GitHub branch protection is all-or-nothing per repo. There's no file-pattern exception. Mixing memory + code + identity in one protected repo creates a structural mismatch.

---

### Recommended architecture

Split by function. Keep the separation clean.

| What | Where | Branch protection | Who reviews |
|------|-------|-------------------|-------------|
| Code changes | Project repo (`agent-engine-dev/src/` etc.) | ✓ Protected | CEO via PR |
| Behaviour files (SOUL, AGENTS, HEARTBEAT, BOOTSTRAP, IDENTITY) | Project repo | ✓ Protected | CEO via PR |
| Config files (TOOLS, USER) | Project repo | ✓ Protected (or optional) | CEO via PR |
| **MEMORY.md** | **Workspace dir only — NOT in project repo** | ✗ None needed | Not reviewed |
| **Session logs (`memory/*.md`)** | **Workspace dir only — NOT in project repo** | ✗ None needed | Not reviewed |

The key insight: **do not wire workspace dirs to the protected project repos.** Keep them separate.

- Project repos = canonical source of truth for code and identity. Protected. CEO-controlled.
- Workspace dirs = operational state only. Not in GitHub at all (or in a separate unprotected location). Agent-controlled.

---

### What this means practically

**For identity file changes** (the audit we just did — SOUL.md, AGENTS.md, HEARTBEAT.md):
- Compass makes changes in the project repos (`/home/pi/idea/agents/agent-{role}/`)
- Compass commits and pushes (to `agent-researcher`, which is unprotected — correct, Compass is CEO-facing)
- Actually: **Compass should not push directly to protected operational repos.** The correct flow is: Compass proposes the changes, CEO reviews, CEO commits to the operational repos, then manually syncs to workspace dirs (or a sync script does it).
- Currently we've been editing workspace dirs directly and not touching project repos at all — that bypasses branch protection but also means project repos are out of date.

**For memory files:**
- Agent commits MEMORY.md and session logs directly in the workspace dir (no PR, no remote needed)
- A nightly backup cron can optionally push workspace memory to a separate, unprotected `memory` repo if historical backup is desired
- The project repo never contains memory files

---

### The sync gap we have right now

After the identity file audit this session, workspace dirs have the new SOUL.md/AGENTS.md/HEARTBEAT.md but project repos don't. The right remediation:

1. Make the changes in the project repos (via PR since they're protected — **but who creates the PR?**)
2. CEO merges the PR
3. Sync the merged files into the workspace dirs

This raises a design question about who owns identity file updates: if only the CEO can merge to protected repos, and operational agents can't create PRs for their own identity files, then identity file maintenance falls to Compass (writing the PR from the researcher repo) or the CEO directly.

**Recommendation:** Treat identity file updates as a Compass responsibility. Compass (this agent, unprotected repo) maintains the canonical identity files in `research/openclaw-initial-config/agent-*/` and stages updates for the CEO to review and apply. The CEO then copies approved files into the project repos. This keeps Compass as the architectural authority without giving agents the ability to self-modify their own governance documents.

---

### Summary

| File type | Needs CEO review? | Why |
|-----------|------------------|-----|
| Code (PRs) | ✓ Yes | Work products; affects software behaviour |
| Identity files (SOUL, AGENTS, HEARTBEAT, BOOTSTRAP, IDENTITY) | ✓ Yes | Governance documents; define agent behaviour and safety rules |
| Config files (TOOLS, USER) | Optional | Infrastructure credentials; rarely change |
| **MEMORY.md** | **✗ No** | **Operational state; must be immediate; not a governance document** |
| **Session logs** | **✗ No** | **Agent's private notes; ephemeral operational data** |

The branch protection model is correct — you just need to keep memory files out of the protected repos entirely, not push them through the PR flow.

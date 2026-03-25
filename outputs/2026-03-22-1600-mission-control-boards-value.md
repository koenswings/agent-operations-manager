> **Question:** What is the added value of Mission Control boards? Detailed feature overview. How does it compare with GitHub PRs? Or with maintaining a tasklist markdown file per agent?

*Based on reading the Mission Control source, README, docs/, and the full OpenClaw gateway API spec (141 endpoints).*

---

## What Mission Control boards actually are

A board is a structured task queue owned by one agent (the lead). Tasks on the board have a status lifecycle (`inbox → in_progress → review → done`), structured metadata, an evidence trail via comments, and optional governance gates (approvals). The board is accessible via a REST API to both the agent and the human operator (via MC UI or CLI).

The key word is *native*: agents read and write boards using the same API you use via the UI. There is no translation layer — the agent's task updates appear in the MC dashboard in real time.

---

## Full feature inventory

### 1. Task management

| Feature | Detail |
|---------|--------|
| Task lifecycle | `inbox → in_progress → review → done` (or `cancelled`) |
| Metadata | Title, description, tags, custom fields, assignee, dependencies |
| Comments | Evidence trail per task — artifacts, decisions, blockers, handoffs |
| Tags | Operator-defined, reusable; agents query them before creating tasks |
| Custom fields | Org-wide additional fields on tasks (defined at org level) |
| Dependencies | Tasks can reference upstream tasks (sequencing) |
| Streaming | Real-time task event streams via SSE |

### 2. Approval governance

| Feature | Detail |
|---------|--------|
| Approval requests | Agent posts `POST /api/v1/agent/boards/{board_id}/approvals` before unsafe actions |
| Board rules | Configurable per board: `require_approval_for_done`, `require_review_before_done`, `only_lead_can_change_status`, `max_agents` |
| Decision trail | Who approved, when, with what note — stored in the approval record |
| Approval streaming | SSE stream of approval state changes |
| Approval discovery | Agent can inspect outstanding approvals before acting |

Current IDEA config: `require_approval_for_done: true` — the CEO must approve every task closure via MC before a task reaches `done`.

### 3. Memory — three tiers

| Tier | Scope | API |
|------|-------|-----|
| Board memory | Scoped to one agent's board | `GET/POST /api/v1/agent/boards/{board_id}/memory` |
| Board-group memory | Shared across a linked group of boards | `GET/POST /api/v1/boards/{board_id}/group-memory` |
| Group memory stream | Near-real-time group memory updates | `GET /api/v1/boards/{board_id}/group-memory/stream` |

Group memory is how cross-agent coordination works without direct agent-to-agent calls. An agent posts to group memory with a `broadcast` or `chat` tag; other agents in the same board group read it at session start. This is how the 2026-03-20 "one task at a time" directive was distributed to all agents.

### 4. Agent coordination

| Feature | API intent | Detail |
|---------|-----------|--------|
| Nudge | `agent_coordination` | Re-engage a stalled or idle agent |
| Direct message | `lead_direct_routing` | Message one specific agent via gateway |
| Broadcast | `lead_broadcast_routing` | Message all board leads at once |
| Human escalation | `human_escalation` | Agent asks user for confirmation via `ask-user` endpoint |
| Roster discovery | `agent_roster_discovery` | Agent discovers all available agents before coordination actions |
| Soul lookup | `agent_board_soul_lookup` | Agent reads another agent's SOUL before deciding how to coordinate |

### 5. Agent lifecycle management

| Feature | Detail |
|---------|--------|
| Heartbeat | Agent signals liveness every ~30s; MC tracks last-seen; up to 3 missed attempts before offline |
| Session management | Operator can inspect active gateway sessions per agent |
| Onboarding workflows | Multi-step guided setup: start → answer → confirm; tracked in MC |
| Agent CRUD | Create, update, delete agents via MC UI or API |
| SOUL authoring | `PUT /api/v1/agent/boards/{board_id}/agents/{agent_id}/soul` — update an agent's role from within MC |

### 6. Board snapshot — the CEO view

`GET /api/v1/boards/{board_id}/snapshot` returns a single aggregated object:
- All tasks by status (inbox count, in_progress count, review count, done count)
- Active agents and their states
- Pending approvals
- Recent board memory

Board **group** snapshot (`GET /api/v1/boards/{board_id}/group-snapshot`) aggregates across all boards in a group — a single API call gives the CEO a view across all 5 agents' boards simultaneously.

### 7. Metrics and dashboards

| Metric | Detail |
|--------|--------|
| KPIs | Task counts by status, WIP, completed in period |
| Time-series | Velocity, WIP by status, burndown |
| Cycle time | Time from task creation to done |
| Period comparison | vs. previous period |

These require enough task throughput to be meaningful. At current IDEA volume they are decorative — they will become useful once tasks are flowing regularly.

### 8. Webhooks

Inbound payload ingestion with audit storage. External systems (CI, GitHub, monitoring) can POST to a webhook endpoint; payloads are stored and can trigger tasks or comments. Currently unused in IDEA but enables:
- CI failure → MC task created automatically
- GitHub PR merged → task status updated

### 9. Skills marketplace

Extensible tool packs that can be installed into agent sessions. Not currently used. Relevant if you want to add capabilities to agents (e.g. a "database query" skill, a "browser" skill) without modifying the core agent config.

### 10. Organization structure

MC supports multi-level hierarchy: Organizations → Board Groups → Boards. For IDEA this maps to: IDEA (org) → [all agents] (board group) → [each agent's board]. The board group enables group memory, group snapshots, and cross-board governance policies.

---

## Comparison: MC boards vs GitHub PRs

These are not alternatives — they solve different problems. But they overlap in one area: **approval before action**.

| Dimension | MC boards | GitHub PRs |
|-----------|-----------|------------|
| **Primary purpose** | Work item tracking + agent coordination | Code review + merge control |
| **What gets tracked** | Tasks (work items, decisions, proposals, reports) | Code diffs |
| **Review unit** | A task + its comment thread | A diff with inline comments |
| **Approval model** | Board rule: `require_approval_for_done` — CEO approves task closure | PR review: CEO approves and merges |
| **Agent-native API** | Yes — agents read/write tasks via REST | No — GitHub API exists but is not designed for agent loop use |
| **Enforcement** | Platform-enforced (can't move to `done` without approval) | Platform-enforced (can't merge without PR approval) |
| **Audit trail** | Per-task comment thread with evidence, decisions, blockers | Per-PR comments, review decisions, CI status |
| **Visibility** | All 5 boards visible in one MC dashboard | GitHub issues/projects exist but separate from agent workflow |
| **CI/CD integration** | Via webhooks (future) | Native (Actions, status checks) |
| **Cross-agent coordination** | Group memory, nudge, broadcast | Not applicable |
| **Metrics** | Velocity, cycle time, WIP | Via GitHub Insights (repo-level, not agent-level) |
| **Code-level review** | No | Yes — diff view, line comments, suggestions |

**Summary:** GitHub PRs are the right tool for *code changes*. MC boards are the right tool for *work items, decisions, and agent coordination*. They are complementary, not competing.

The current IDEA model uses both correctly: code changes go through GitHub PRs; task-level planning and approval uses MC boards. The auto-review protocol bridges them: when Axle opens a PR, it creates a review task on Veri's MC board. Veri reviews the PR and comments on the task. CEO sees both in their respective systems.

---

## Comparison: MC boards vs a markdown tasklist per agent

This comparison is more direct — both are ways to track what an agent is doing.

| Dimension | MC boards | Markdown tasklist (`TASKS.md`) |
|-----------|-----------|-------------------------------|
| **Infrastructure** | MC stack must be running | None — just a file |
| **Agent access** | REST API (structured, typed) | File read/write (freeform) |
| **Enforcement** | Board rules enforced by platform | None — agent can ignore it |
| **Approval gates** | Platform-enforced (`require_approval_for_done`) | Convention only |
| **CEO visibility** | Dashboard — all 5 agents at once | Must open 5 files |
| **Audit trail** | Task-level comment history with timestamps | Git log (commit-level, not task-level) |
| **Cross-agent tasks** | Agent B creates task on Agent A's board via API | Agent B edits Agent A's TASKS.md — messy |
| **Real-time** | SSE streams for task/approval events | None — polling the file |
| **Task metadata** | Tags, custom fields, dependencies, assignees | Whatever you put in the markdown |
| **Metrics** | Built-in velocity/cycle-time | None |
| **Reliability** | Depends on MC stack uptime | Always available |
| **Failure mode** | MC down → agents can't update board | File always readable/writable |
| **Complexity** | High: board IDs, API calls, auth tokens | Zero |

**What MC boards add over a markdown file:**
1. **Enforcement** — the CEO approval gate before `done` cannot be bypassed. A markdown checklist can be rewritten by the agent.
2. **CEO dashboard** — one view across all 5 agents. With markdown files you navigate to 5 different directories.
3. **Cross-agent review protocol** — the auto-review pattern (Axle creates task on Veri's board, cron triggers Veri) cannot be replicated cleanly with markdown files without building custom machinery.
4. **Audit trail** — task comments carry timestamps, actor, and evidence. Git history on a markdown file tells you the file changed, not which task was worked and what the outcome was.

**What a markdown file does that MC boards don't:**
1. **Always works** — no stack dependency, no API, no auth. If the Pi reboots and MC is not restarted (current gap), tasks disappear from view. The file is always there.
2. **Human-readable in context** — the agent can reason about a markdown task list in the same flow as reading SOUL.md and MEMORY.md. Board access requires separate API calls.
3. **Zero operational overhead** — no board IDs to track, no API calls, no heartbeat.
4. **Survives a full system rebuild** — a markdown file in a git repo is preserved. A MC board lives in a Docker volume and is lost if the volume is deleted.

---

## Honest assessment for IDEA right now

MC boards are the right architectural choice for IDEA at scale. The approval gates, cross-agent review protocol, and CEO dashboard are genuine value that a markdown file cannot provide without rebuilding the same infrastructure yourself.

But two gaps are making the current setup feel more complex than it is:

**Gap 1: MC has no restart policy.** The MC stack stops when the Pi reboots or crashes. When MC is down, agents cannot update their boards. This is the single most important operational fix — it makes the system feel unreliable. Fix: add `restart: unless-stopped` to all services in the MC compose.yml.

**Gap 2: No tasks are flowing.** The boards have tasks in inbox but none are being picked up. A board system with no activity provides zero value over a markdown file. The value of MC becomes visible only when the CEO-invokes-agent → agent-works-task → CEO-approves loop is actually running.

The complexity you are feeling is mostly from setup state, not from the system being wrong. The architecture is sound. The missing pieces are: MC restart policy, workspace consolidation (code repo as workspace), and the first real task cycle being run end-to-end.

---

*Compass · Strategic Advisor · IDEA*

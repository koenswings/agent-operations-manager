> **Question:** Can we reduce complexity by switching Compass to OpenClaw/Telegram? What is the difference between managing Compass via Claude directly versus via OpenClaw? How does the memory model differ? Are there other differences? Does Mission Control add anything (e.g. boards with tasks)? What are boards with tasks?

---

## Claude direct vs OpenClaw — what actually differs

---

### The short answer

You are right that this is a two-system mess. The complexity is real. Moving Compass to OpenClaw is low-effort and would unify the access model — and Compass is **already partially configured for it** (it has an entry in `openclaw.json`). Whether it is worth it depends on what you gain and what you lose, which this document unpacks.

---

### What each system is

**Claude direct (current Compass model)**

You open a terminal (tmux/VSCode), navigate to the `agent-researcher` directory, and run `claude`. Claude Code is the runtime — it is an interactive CLI session. CLAUDE.md is the identity/role instruction. Auto-memory files in `~/.claude/projects/.../memory/` are injected into every session automatically.

The session ends when you close the terminal. There is no background process. There is no API.

**OpenClaw**

OpenClaw is a gateway layer that wraps the Claude API. It runs as a Docker container on the Pi. When a message arrives (from MC chat or Telegram), the gateway spins up a Claude Code subprocess in the agent's workspace dir, passes the message, captures the output, and posts it back. Then the subprocess exits.

The agent is not "running" between messages — it is invoked on demand, same as Claude direct, but the invocation is mediated by OpenClaw rather than by you opening a terminal.

---

### The memory model — where they actually differ

This is the main structural difference.

#### Claude direct memory

| Property | Detail |
|----------|--------|
| Managed by | Claude Code itself |
| Location | `~/.claude/projects/-home-pi-idea-agents-agent-researcher/memory/` |
| Loading | **Automatic** — MEMORY.md is injected into system context at session start, every time |
| Writing | Claude writes to memory files during a session using the Write tool |
| Format | MEMORY.md is an index; typed files (user, feedback, project, reference) linked from it |
| Backup | None — local Pi filesystem only |

**Key property:** Claude Code manages the memory lifecycle. You do not need to instruct Compass to read its memory — it is already in context before the first message.

#### OpenClaw memory

| Property | Detail |
|----------|--------|
| Managed by | The agent itself, as instructed |
| Location | Inside the workspace dir: `memory/YYYY-MM-DD.md` + `MEMORY.md` |
| Loading | **Manual** — AGENTS.md instructs the agent to read `MEMORY.md` and today's date file at session start |
| Writing | Agent writes memory files using file tools during the session |
| Format | Daily log files + durable `MEMORY.md` for decisions/standards; no enforced structure |
| Backup | None — local Pi filesystem only (workspace dirs are not on GitHub) |

**Key property:** The agent reads memory because it is instructed to, not because the platform injects it. The workspace dir is the filesystem the Docker container sees, so the files are directly accessible.

#### What is actually different between the two

| | Claude direct | OpenClaw |
|--|--------------|----------|
| Memory injection | Automatic (platform-level) | Manual (instruction-level) |
| Memory location | `~/.claude/` (outside repo) | Inside workspace dir (inside repo tree) |
| Memory structure | Typed, indexed format | Free-form daily logs + durable doc |
| Who controls format | Claude Code system prompt | You (via AGENTS.md) |

In practice: both are file-based, both are manual (Claude writes, Claude reads), and neither has remote backup. The OpenClaw model puts memory **inside the mounted workspace** — so it is visible, inspectable, and survives as long as the Pi filesystem does. The Claude direct model stores memory in a hidden `~/.claude/` path you cannot easily see without navigating there.

**OpenClaw memory is actually more transparent and more auditable.** You can open the workspace dir in a file browser and read every memory file. The Claude direct memory is tucked away in an internal path most users never look at.

---

### Other differences beyond memory

#### 1. Session trigger

| | Claude direct | OpenClaw |
|--|--------------|----------|
| How a session starts | You open a terminal and run `claude` | A message arrives (from MC or Telegram); OpenClaw invokes Claude automatically |
| Access surface | Terminal on the Pi (or SSH) | Any device — web browser (MC), phone (Telegram) |
| Scheduling | Not possible without a cron wrapping it | Heartbeat schedule built in (though currently disabled for all agents) |

#### 2. Interface

Claude direct outputs text to the terminal, which gets truncated in small VSCode windows (which is why every response gets written to a file instead). OpenClaw outputs to the MC chat UI or Telegram — both handle long responses correctly.

**This is the single biggest operational improvement of moving Compass to OpenClaw.** The "write everything to a file because the terminal is too small" workaround goes away. Compass's response appears directly in the MC chat or Telegram thread, readable on any device.

#### 3. Identity and instruction layer

| | Claude direct | OpenClaw |
|--|--------------|----------|
| Role instructions | `CLAUDE.md` | `SOUL.md`, `AGENTS.md`, `IDENTITY.md` |
| Surviving reprovision | `CLAUDE.md` always there (in git) | Only `USER.md` and `MEMORY.md` are preserved on forced reprovision; SOUL.md is overwritten |
| Flexibility | Edit one file in the repo | Richer split (persona, role, tools, user profile are separate files) |

#### 4. Permission model

| | Claude direct | OpenClaw |
|--|--------------|----------|
| Default mode | `bypassPermissions` (set in settings.local.json) | `plan` (set via `DEFAULT_PERMISSION_MODE=plan` in compose.yaml) |
| Overridable per session? | Yes (CLI flag or settings) | Yes (via gateway config) |

Compass runs with no prompts. OpenClaw agents run in plan mode — they propose before acting, CEO approves.

If Compass moved to OpenClaw, it would inherit plan mode by default (as all OpenClaw agents do). This could be configured off — the SOUL.md equivalent could note that Compass should proceed without plan mode prompts since it only writes files, not code. But you would need to explicitly configure this.

#### 5. Multi-agent coordination

OpenClaw has **group memory** — a shared memory bus across agents with tagged entries (chat, broadcast, etc.). Operational agents can post to group memory when they need to broadcast a decision across all agents. Claude direct has no equivalent — Compass is fully isolated.

This is correct for Compass as a strategic advisor (isolation is intentional). If Compass moved to OpenClaw, it would technically gain access to group memory, but it should not use it — the isolation rule is instructional, not a platform feature.

---

### What Mission Control adds

Mission Control (MC) is a separate web app that sits on top of the OpenClaw gateway. It is the UI layer.

**Without MC:** You interact with agents only via the gateway's own interface (or Telegram bindings).

**With MC:** You get:

| Feature | What it does |
|---------|-------------|
| Chat UI per agent | A web chat interface for each agent; persistent message history; accessible in browser |
| Boards | A task management system per agent (described below) |
| Agent management | View all agents, their status, provision/deprovision |
| Activity timeline | Audit log of all system events — what ran, when, what it produced |
| Gateway management | Connect/inspect remote gateways |
| Approvals | Route sensitive task closures through explicit approval flows |

---

### What are boards and tasks

Every operational agent has a **board** — think of it as the agent's to-do list, managed via an API that both the agent and MC can read and write.

**Structure:**

```
Board (per agent)
  └── Tasks
        ├── status: inbox | in_progress | review | done
        ├── title + description
        ├── tags (e.g. "auto-review")
        ├── comments (evidence trail)
        └── approvals (required before done, if board rule set)
```

**How tasks flow:**

1. CEO creates a task on an agent's board via MC UI (or API), or an agent creates a review task on another agent's board.
2. Agent picks up the task, moves it to `in_progress`, and works on it.
3. Agent posts comments with evidence as it works (not status spam — concrete artifacts only).
4. When done, agent moves task to `done`. If `require_approval_for_done: true` (current rule for all agents), MC shows the task as pending CEO approval.
5. CEO approves. Task closes.

**Board rules (per Axle's current config):**

| Rule | Value | Effect |
|------|-------|--------|
| `require_approval_for_done` | `true` | CEO must approve every task closure |
| `require_review_before_done` | `false` | No mandatory review gate (agents handle review themselves via auto-review protocol) |
| `only_lead_can_change_status` | `false` | Any agent can move tasks |
| `max_agents` | `1` | Only the lead agent works this board (no specialist sub-agents currently) |

**The auto-review protocol** is built on top of boards: when an agent completes primary work, it creates a review task on the reviewer's board (tagged `auto-review`). A Pi cron detects this tag and triggers the reviewer agent. The reviewer reads the artifact, posts a comment, marks the task done. This is agent-to-agent coordination via the board API — no human routing needed.

**What boards add that Telegram alone does not:**
- Persistent task queue — tasks survive across sessions; nothing gets lost in a chat stream
- Approval gates with audit trail
- Cross-agent review workflow (the auto-review protocol)
- CEO dashboard view across all agents' boards at once

**What boards add that a GitHub project board does not:**
- Native integration with the agents — they read and write tasks via API, post evidence in comments, trigger each other
- Governance rules enforced by the platform (not by convention)

---

### Should Compass move to OpenClaw?

**What you gain:**
- Access from any device — browser (MC) or phone (Telegram), no terminal needed
- Long responses visible in chat, no truncation workaround, no forced file writes
- Memory stored in the workspace dir (transparent, inspectable) rather than `~/.claude/`
- Consistent access model across all 6 agents
- Compass already has an `openclaw.json` entry — foundation is there

**What you lose or must add:**
- The auto-memory injection that Claude Code does automatically. OpenClaw agents rely on AGENTS.md instructing them to read memory. Compass would need an equivalent AGENTS.md-style file (or the CLAUDE.md can serve this role if OpenClaw mounts the workspace at the same path).
- The `bypassPermissions` auto-approval would be replaced by `plan` mode by default. This is configurable but needs a deliberate decision.

**What does not change:**
- The work Compass does: research, writing documents, reading files across the org
- The output location: `outputs/` and `research/` directories in the same workspace
- The isolation from operational agents: this is instructional, not platform-enforced

**The underlying complexity being complained about** is not really Claude-direct vs OpenClaw — it is the two-tier setup where the workspace dirs live separately from the project repos, memory has no remote backup, and there are five separate access surfaces (five Telegram groups, one MC per agent, one terminal per agent). That complexity exists in the OpenClaw layer regardless of what Compass does.

Moving Compass to OpenClaw makes the **access model simpler** (one system instead of two), but does not resolve the underlying retention gap or the workspace/repo split.

---

*Compass · Strategic Advisor · IDEA*

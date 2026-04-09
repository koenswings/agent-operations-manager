# MC Deeper Integration — Design Exploration

**Status:** Draft  
**Date:** 2026-04-09  
**Author:** Atlas  
**Type:** Exploratory — no decisions implied

---

## Why this exists

The current workflow keeps MC as a pure task register and runs all
communication through Telegram. That's a deliberate, working choice.
But there's a real tension: task-threaded comments in MC are
genuinely better than a single Telegram thread when you're managing
work across multiple agents at once — parallel conversations stay
separate, context doesn't bleed. And managing task state (Inbox →
In Progress → Review → Done) from the MC UI is clean when you're at
a desk, but requires opening a browser when you're on the go.

This document explores what it would look like to shift more of the
workflow into MC — and where Telegram still wins. It is not a
proposal for immediate action. It is a map of the option space.

---

## Part 1 — Full MC UI Reference

Before comparing routes, a complete picture of what MC actually
offers. Each section covers a feature, what it does technically, and
an IDEA-specific assessment.

---

### 1.1 Boards and board groups

**What it is:** Boards are the top-level unit of work per agent.
Board groups (Engineering, HQ) organise boards. The sidebar shows
groups; clicking a group reveals its boards; clicking a board shows
its Kanban.

**Current state:** 5 boards, 2 groups. Every board has
`require_approval_for_done: true` — Done requires CEO action.

**How to use it:** The group view is the morning dashboard. Open
Engineering → scan all four boards at once for anything in Review.
Open HQ for Marco. That's the full operational picture in two clicks.

**IDEA assessment:** This is already the right setup. No changes
needed here.

---

### 1.2 Task cards — the Kanban columns

**What it is:** Four columns per board: `inbox`, `in_progress`,
`review`, `done`. Tasks are cards; drag to move. Click to open.

**Full task schema:**
- `title` — required, short action phrase
- `description` — markdown, full context for the agent
- `status` — inbox / in_progress / review / done
- `priority` — low / medium / high (default: medium)
- `due_at` — optional deadline (ISO timestamp)
- `depends_on_task_ids` — task dependency chain
- `tag_ids` — coloured labels (see §1.6)
- `custom_field_values` — org-level custom fields (see §1.7)
- `assigned_agent_id` — which agent owns it

**Moving tasks:**
- Drag card to column, or open task → change Status dropdown
- When moving to Done: requires CEO approval (board setting)
- TaskUpdate also accepts an optional `comment` field — you can
  log a note at the same time as changing status

**IDEA assessment — status management from Telegram:**
This is the main friction point. When you're mobile, moving a task
to Done means opening a browser tab. See §2 (Telegram integration
routes) for how Atlas could handle this with a simple Telegram
command.

**Dependencies:** `depends_on_task_ids` is genuinely useful for the
IDEA backlog. Many Axle tasks are sequenced (you can't implement
copyApp before ejectDisk). Setting dependencies makes that visible
in the UI and prevents accidentally marking a blocked task in
progress.

---

### 1.3 Task comments

**What it is:** Every task has a comment thread. Any user or agent
can post. Comments are timestamped and permanent.

**Why this is better than Telegram for task communication:**
- Each task has its own thread — parallel conversations on 3 tasks
  stay separate
- Context is attached to the task, not to a chat session — survives
  session restarts, agent wakeups, weeks-long gaps
- When Axle wakes up on a task two weeks later, the comments are
  right there on the card
- You can write a comment at 11pm when something occurs to you,
  without it triggering an immediate agent response

**Current gap:** Agents don't actively monitor MC comments. They
read `BACKLOG.md` at session start, which only exports task titles
and statuses — not comments. If you write a task comment, the agent
won't see it until you also message them in Telegram.

**Fix options (see §2):** Either (a) agents read task comments at
session start for their in-progress tasks, or (b) Atlas posts a
comment notification to the agent's Telegram when you add a comment.

---

### 1.4 Approvals

**What it is:** A formal approval mechanism separate from task
status. An agent posts an approval request (action_type, payload,
confidence score, reasoning) before taking an irreversible action.
You see it in the MC UI, approve or reject, the agent proceeds or
aborts. The decision is logged permanently.

**Schema:**
- `action_type` — e.g. "send_external_email", "wipe_disk_partition"
- `payload` — structured description of what will happen
- `confidence` — agent's self-reported confidence (0–100)
- `rubric_scores` — optional per-criterion scores
- `lead_reasoning` — agent's explanation
- `status` — pending / approved / rejected

**Current use:** Not active. All approval happens through Telegram
plan-mode (agent shows plan → you type yes/no).

**IDEA assessment — when this adds value:**
Telegram plan-mode is sufficient for code changes (there's always a
PR as the second gate anyway). Approvals add value for one-shot
irreversible actions with no git trail:
- Marco about to send a briefing email to an external funder
- Axle about to wipe a disk partition
- Marco about to post to the IDEA supporters WhatsApp group

For these, the structured approval record in MC is better than a
"yes" in Telegram that disappears in chat history.

**Not worth activating for:** Routine PR-based work. The PR is the
approval record. Don't add MC approval ceremony on top.

---

### 1.5 Board memory

**What it is:** A per-board key-value store, separate from git.
Agents can write durable operational state (e.g. "current
architecture decision", "last reviewed PR"). Visible in the MC UI.
Tags and source fields available.

**Group memory:** A shared memory space visible to all boards in a
group. Cross-agent shared state without going through git.

**IDEA assessment:** Low priority now. The git-based `MEMORY.md`
files do this job and have version history. Board memory would
complement (not replace) them if agents start using MC more heavily.
One concrete use case: Axle writes the current Engine architecture
decision to board memory so it's visible in the MC UI alongside the
task board — without requiring a git commit.

---

### 1.6 Tags

**What it is:** Coloured labels attached to tasks. Created at org
level, usable on any board.

**Schema:** name, slug, color (hex), description.

**IDEA assessment — useful taxonomy:**
A small tag set would add visual clarity to the IDEA boards:

| Tag | Color | Use |
|-----|-------|-----|
| `blocked` | red | Task waiting on external dep |
| `needs-design` | orange | Requires design doc first |
| `pr-open` | blue | PR exists, in review |
| `field-impact` | green | Affects school deployment |
| `cross-agent` | purple | Needs input from another agent |

Not essential — but once the boards have 15+ tasks, tags make
the Kanban scannable at a glance.

---

### 1.7 Custom fields

**What it is:** Org-level custom fields attached to tasks. Flexible
key-value data beyond the default task schema.

**IDEA assessment:** Probably not needed at current scale. The
description field handles task context. Custom fields would become
relevant if you want structured metadata — e.g. "PR URL", "affected
Pi count", "school name" — that you want to filter or report on.
Park for later.

---

### 1.8 Webhooks

**What it is:** Per-board inbound webhooks. External services POST
events to MC; MC stores the payload and can route it to the agent.

**IDEA assessment — the GitHub PR integration:**
This is the most valuable underused feature. GitHub can POST PR
events to MC:
- PR opened → task moves to Review automatically
- PR merged → task moves to Done automatically

This closes the loop between GitHub and MC without manual task
management. Implementation:
1. Create a webhook on the Engine Dev board
2. Add a GitHub Actions step in `agent-engine-dev` to POST to the
   MC webhook URL on PR events
3. Atlas (or a small script) maps PR numbers to task IDs and calls
   the MC API to update status

This is a medium-effort improvement that makes MC dramatically more
useful. Without it, task state in MC always lags behind GitHub.

---

### 1.9 Nudge

**What it is:** Atlas can POST a nudge message to any board lead.
The message is injected into the agent's active gateway session as
an immediate directive: "Please update the incident triage status
for task T-001."

**IDEA assessment:** This is Atlas's tool for cross-agent
coordination without going through Telegram relay. Currently
the relay is: Atlas sends `📨 For Axle: [message]` in its own
Telegram group → Koen reads it → forwards to Axle's group. Nudge
could replace the forwarding step for simple directives.

**Caveat:** Still requires Koen to initiate the nudge (Atlas can't
send it autonomously without approval). The value is that Koen could
approve a nudge in MC rather than re-typing the message in Telegram.
Not a major gain at current scale.

---

### 1.10 Ask-user

**What it is:** An agent can POST a blocking question to the CEO via
the gateway main channel. The question appears in Telegram (or
wherever the main session is bound). The agent waits for the answer.
The exchange is logged in MC.

**Schema:** content (the question), preferred_channel, correlation_id
for tracing, reply_tags.

**IDEA assessment:** This is a cleaner version of the current
Telegram plan-mode flow — but logged. An agent that needs a
decision before proceeding can ask via MC rather than through an
ad-hoc Telegram message. The difference: the question is attached
to a task, so context is preserved.

Worth activating for agents once they're using MC more heavily.

---

### 1.11 Gateway sessions

**What it is:** MC can list all active OpenClaw gateway sessions,
fetch their history, and inject messages into them directly via the
API (`POST /api/v1/gateways/sessions/{session_id}/message`).

**IDEA assessment:** This is the foundation of "MC as orchestration
layer." If MC knows the session ID for each agent, it can message
any agent directly without going through Telegram. This is not
currently wired up (agent session IDs in MC DB were stale until
Claude fixed them yesterday). Now that they're correct, this is
live infrastructure that just needs to be used.

**Concrete use:** Atlas could use this to relay cross-agent messages
without requiring Koen to forward manually. An Atlas cron checks for
`cross-agent` tasks on other boards and fires session messages
directly. This is the "MC board tasks (future)" path described in
`virtual-company-design.md`.

---

### 1.12 Activity feed / streaming

**What it is:** A real-time SSE stream of task comment activity
across all boards (`GET /api/v1/activity/task-comments/stream`).
Visible passively in the MC UI.

**IDEA assessment:** Useful for monitoring during active sessions.
If you have MC open on a desktop tab while working, you see all
agent comments appear live without refreshing. Low-effort passive
visibility.

---

### 1.13 Dashboard metrics

**What it is:** Aggregate usage stats per board or group, over a
configurable time range (24h to 1y). Likely shows task throughput,
completion rate, agent activity.

**IDEA assessment:** Not useful at current scale. Revisit when
boards have meaningful historical data (6+ months of tasks).

---

### 1.14 Agents panel and heartbeat

**What it is:** MC tracks agent registration, health status, and
last heartbeat time. Agents can POST heartbeats to report they're
alive and active. The UI shows online/offline status.

**Current state:** All five agents show "healthy" but heartbeats
are not being sent. Status is static.

**IDEA assessment:** Agents sending heartbeats when they complete a
task or session would make the "all agents online" indicator
meaningful. Low priority — more informational than operational.

---

### 1.15 Skills marketplace

**What it is:** MC has its own skill install system, separate from
the `/skills/` folder in the idea repo. Skills can be listed,
installed, and uninstalled via API.

**IDEA assessment:** The relationship between MC skills and IDEA's
`/skills/` directory is unclear. Don't invest in this until the
distinction is understood. Park.

---

### 1.16 Souls directory

**What it is:** A searchable directory of agent soul templates.
Atlas could read any agent's SOUL equivalent without filesystem
access.

**IDEA assessment:** Not relevant for IDEA — SOUL.md files are
managed locally and backed up to `agent-identities`. The souls
directory would only matter in a multi-tenant cloud deployment.

---

## Part 2 — Integration Routes

Three distinct routes, ordered from least to most change. Each is
implementable independently — they're not a sequence.

---

### Route A — "Telegram-first, MC as record"

**The minimal shift. Current behaviour + two additions.**

**Changes:**
1. Agents read their in-progress task comments at session start
   (not just task titles from BACKLOG.md)
2. Atlas posts a Telegram notification when you add a comment to
   any task — "💬 Comment on [task title]: [first 80 chars]"

**Result:** Task comments become a real async communication channel.
You write a comment on a task at any time; the agent sees it at
next wakeup without you having to repeat yourself in Telegram.
Telegram remains the primary surface; MC is the structured record.

**Telegram stays:** All conversation, plan approval, task
assignment notifications, PR feedback.  
**MC gains:** Persistent task-threaded comments that agents actually
read.

**Effort:** Low. Changes to each agent's AGENTS.md startup checklist
+ one Atlas cron job for comment notification.

**What you lose:** Nothing from current workflow.

---

### Route B — "Dual surface, by context"

**MC for desk work; Telegram for mobile. Atlas bridges the gap.**

**The core insight:** MC is better for structured work (reviewing
tasks, writing detailed comments, approving Done). Telegram is
better for mobile, quick questions, and things that need an
immediate response. You don't have to pick one — you operate both
and Atlas keeps them in sync.

**Changes beyond Route A:**
1. **Task state via Telegram commands** — Atlas handles a small
   command vocabulary so you can manage MC state from Telegram
   without opening a browser:
   - `done [task keyword]` → Atlas finds the task, moves to Done
   - `review [task keyword]` → moves to Review (or prompts for
     which task)
   - `comment [task keyword] [text]` → posts a comment to the task
   - `tasks axle` → Atlas replies with Axle's current board state
     as a brief text list
2. **GitHub → MC webhooks** for Engine Dev, Console Dev, Site Dev
   — PR opened sets task to Review; PR merged sets to Done. Task
   state tracks reality automatically.
3. **Approvals for irreversible actions** — Marco sending external
   communications, Axle destructive disk ops. MC approval request
   appears in Telegram via ask-user; you approve in Telegram; MC
   logs it.

**Result:** MC and Telegram are both fully operational. You use
whichever is convenient. Task state is accurate without manual
updates because GitHub webhooks handle the transitions
automatically.

**Effort:** Medium. Telegram command parser in Atlas's session
(triggered by message patterns) + GitHub Actions webhook integration
+ enabling MC approvals for Marco and Axle.

**What you lose:** Nothing. Pure addition.

---

### Route C — "MC as primary, Telegram as notification layer"

**The full shift. All task management in MC; Telegram becomes
read-mostly.**

**Changes beyond Route B:**
1. **Agents use ask-user** for all blocking questions — routed to
   Telegram via gateway main, but logged in MC with full context
2. **Atlas cross-agent coordination via gateway sessions** — the
   Telegram relay for `📨 For [Agent]` messages is replaced by
   Atlas directly injecting messages into agent sessions via the
   MC Gateways API. You still approve in Telegram, but don't need
   to forward manually.
3. **Board memory as operational state** — agents write current
   decisions and architecture state to MC board memory; visible
   in the UI alongside tasks
4. **Telegram becomes notification-only** — agent updates come
   through ask-user and the activity feed; you respond in MC
   (approvals, comments) rather than Telegram chat

**Result:** MC is the operational centre. Telegram pings you when
something needs attention; you handle it in MC. Sessions with agents
are for complex direction-setting, not routine status updates.

**Effort:** High. Significant changes to every agent's AGENTS.md
and session behaviour. New infrastructure (ask-user routing, session
ID management, cross-agent nudge flow).

**What you lose:** The conversational fluidity of Telegram. MC is
structured but less natural for open-ended direction. The "talk to
your agent" feeling diminishes.

---

## Part 3 — Recommendation

Start with **Route A**. It's two small changes that make task
comments actually useful. No workflow disruption. Ship it in one PR.

Evaluate **Route B elements** independently:
- GitHub webhooks for PR status sync → high value, medium effort,
  implement when the boards have active tasks with open PRs
- Telegram task commands → useful when you're regularly moving
  tasks to Done from mobile; implement if the friction of opening
  a browser for that step becomes annoying
- MC approvals for Marco → implement when Marco starts sending
  external communications

**Don't pursue Route C yet.** The conversational Telegram workflow
works well for a solo CEO. Route C makes sense when the team grows
or when you want a structured audit trail that's not possible via
Telegram logs.

---

## Part 4 — What this would NOT change

Regardless of route:
- GitHub remains the gate for all code changes — PRs and merges
  stay there
- Telegram remains the channel for real-time conversation and
  things that need an immediate response
- The CEO approval loop is preserved in all routes — agents never
  act unilaterally
- `BACKLOG.md` continues to be the git-traceable snapshot that
  agents read at session start — MC is not the only source of truth

---

## Open questions before implementing any route

1. **Comment visibility:** Does the BACKLOG.md export need to
   include in-progress task comments, or is a separate
   "read my task comments" step at session start sufficient?
2. **Telegram command surface:** How should Atlas identify which
   task you mean when you type `done [keyword]`? Fuzzy match on
   title? Or always list candidates and confirm?
3. **GitHub webhook auth:** MC webhooks support a secret + signature
   header. What's the right secret management approach — stored in
   `.env`, or in platform secrets?
4. **Session ID stability:** The MC DB now has correct agent session
   IDs. Are these stable across gateway restarts, or does each
   session start generate a new ID? If unstable, the gateway session
   message approach needs a lookup step first.

# MC Deeper Integration — Design Exploration

**Status:** Closed — decision reached 2026-04-11  
**Date:** 2026-04-09  
**Author:** Atlas  
**Type:** Historical reference — kept for context, not for action

---

## ⚠️ Decision (2026-04-11)

After reviewing this document, Koen decided: **we will not expand MC usage beyond what we do today.**

MC continues as a task register — boards, task status, and agent health visibility. No further integration routes will be pursued.

This document is preserved for historical reference. None of the routes (A, B, C) are planned. Do not use this as a basis for proposing MC feature work.

---

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

---

## Part 5 — What the Authors Say: Channels, MC, Backup, and Installation

A comparison between documented intent and our actual setup.
Sources: OpenClaw docs (docs.openclaw.ai), MC docs (/home/pi/openclaw/mission-control/docs/),
OpenClaw README (/home/pi/openclaw/README.md).

---

### 5.1 Chat channels and their role in MC

**What the OpenClaw docs say:**

OpenClaw's channel documentation describes Telegram as the fastest
channel to set up — just a bot token, no QR or phone number — and
as production-ready for both DMs and groups via the grammY library.
The docs describe channels as independent of MC. MC is a separate
layer on top — a control plane for boards, tasks, and approvals.
The docs don't describe any native integration between Telegram
messages and MC task management. They are presented as two distinct
ways to interact with the gateway, not a unified surface.

On group chats: the docs describe per-group session isolation
(`session key: agent:<agentId>:telegram:group:<chatId>`). Each
Telegram group gets its own session, separate from DMs. Commands
like `/verbose on` sent in a group apply only to that group's
session. This is exactly what IDEA relies on — each agent has its
own group with an isolated session.

On access control: the recommended pattern for a single-owner bot
is `dmPolicy: "allowlist"` with your numeric user ID in `allowFrom`,
plus target groups listed under `channels.telegram.groups`. The
docs flag a subtle security boundary: DM pairing approval does not
grant group access. Group sender auth is controlled separately via
`groupAllowFrom` or the fallback `allowFrom`.

On MC + channels: the MC docs describe MC as the "day-to-day
operations surface" with gateway connections as a first-class
feature. The ask-user endpoint routes questions to "the gateway
main channel" — which is whatever channel the gateway is bound to
(in our case Telegram). So the integration point the MC authors
had in mind is ask-user → gateway main session → your Telegram.
Nothing in the MC docs describes task state management via Telegram
commands. That's a pattern we'd be building ourselves.

**What we do:**

We use exactly the recommended Telegram setup: bot token in
`openclaw.json`, `dmPolicy: "allowlist"` with numeric sender IDs,
one group per agent. This is textbook. The only non-standard element
is that we use groups (one per agent) as the primary interface
rather than DMs — the docs present DMs as the default interaction
model and groups as an opt-in extension. Our approach is supported
but not the primary path the docs describe.

MC and Telegram are currently entirely separate in our setup, which
aligns with how both tools are documented. The ask-user bridge is
unused — all blocking questions happen through Telegram plan-mode,
not routed through MC.

**Gap:**

There is a documented but unused integration path: agent → MC
ask-user → gateway main → your Telegram. This is the author-intended
bridge between MC and messaging channels, and we haven't wired it
up. The MC docs also describe the `send_gateway_session_message`
endpoint as requiring org-admin membership, which means it's
intentionally gated — not something agents should call freely.

---

### 5.2 MC backup and state persistence

**What the MC docs say:**

The MC operations runbook (docs/operations/README.md) gives explicit
backup instructions:

> The DB runs in Postgres (Compose `db` service) and persists to
> the `postgres_data` named volume.
>
> Example with `pg_dump` (run on the host):
> ```bash
> PGPASSWORD="$POSTGRES_PASSWORD" pg_dump \
>   -h 127.0.0.1 -p "$POSTGRES_PORT" -U "$POSTGRES_USER" \
>   -d "$POSTGRES_DB" --format=custom > mission_control.backup
> ```

The docs note this is a minimal backup and recommend automated
backups with retention and periodic restore drills for real
production.

The docs also describe a token re-sync procedure for after reinstalls
(`POST .../gateways/GATEWAY_ID/templates/sync?rotate_tokens=true`)
in the gateway agent provisioning troubleshooting guide. This implies
the authors expected reinstalls to happen and built a recovery path —
but it assumes MC is still running and you're syncing tokens from
MC back to the gateway. It does not cover the case where both MC
and OpenClaw are wiped and rebuilt from scratch.

**What we do:**

We back up nothing in MC. The nightly backup cron copies agent
identity and memory files from the filesystem to `agent-identities`
on GitHub. The MC PostgreSQL database — which holds all boards,
tasks, task comments, agent records, approvals, and the gateway
configuration — is not backed up anywhere.

This is a significant gap. If the Pi's SD card fails or we do a
fresh `setup.sh` install, we lose:
- All task history across all five boards
- All task comments
- All approval records
- The gateway DB entry (URL, token, `disable_device_pairing`)
- All agent registrations with their board IDs and session IDs
- The `NEXT_PUBLIC_API_URL` and other env that was set manually

The `setup.sh` script provisions new MC boards and agent tokens on
fresh install, so the boards themselves would be recreated — but
empty. Task history is permanently lost.

**What we should do:**

Add a `pg_dump` step to the nightly backup script alongside the
identity file backup, writing a compressed dump to a known path
that the `backup-agent-identities.sh` then pushes to GitHub (or
a separate private repo). The dump is small — tasks and comments
are text, not binary — and a daily snapshot is sufficient for IDEA's
use pattern.

A restore path should also be documented: `pg_restore` the dump
after `setup.sh` brings up the stack, then re-run
`templates/sync?rotate_tokens=true` to re-link gateway tokens.

This should be treated as a P1 gap, not a future-phase item.

---

### 5.3 OpenClaw installation: docs vs. our approach

**What the docs say:**

The canonical install path is a one-liner:
```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```
or `npm install -g openclaw@latest` followed by
`openclaw onboard --install-daemon`.

The onboarding wizard handles: model provider selection, API key,
gateway config, channel setup, and systemd service install.
The docs describe this as a ~5 minute process. Config lives at
`~/.openclaw/openclaw.json`, which the gateway watches and hot-reloads
on change. The docs offer a `openclaw configure` CLI wizard and a
Control UI config form as alternatives to hand-editing JSON.

For Telegram specifically: configure `botToken` in `openclaw.json`,
start the gateway, then `openclaw pairing approve telegram <CODE>`
for first-time DM access. No Docker involved. No proxy required.
The docs present the native install as the standard path; Docker
is documented as a container deployment option, not the default.

**What we do:**

We use `npm install -g openclaw@latest` (via `setup.sh`) — which
matches one of the documented paths exactly. The config lives at
`~/.openclaw/openclaw.json` and the systemd service is installed
via `openclaw gateway install` — also exactly as documented.

**Where we diverge:**

1. **No onboarding wizard used.** We bypass `openclaw onboard` and
   instead provision everything programmatically via `setup.sh`. The
   wizard would configure the same things, but the script gives
   reproducibility. This is a deliberate and reasonable deviation —
   the wizard is for humans, `setup.sh` is for automation.

2. **The old Docker-in-Docker era left residue.** The `README.md`
   in `/home/pi/openclaw/` documents a Docker-based architecture
   with a TCP proxy in `entrypoint.sh`, a Tailscale container, and
   `openclaw.json` stored in a Docker volume. This was the original
   IDEA setup before the native migration (April 2026). That
   architecture is gone — OpenClaw now runs natively — but the
   README still documents the old Docker approach. It's a stale
   artefact, not the live configuration.

3. **We manage `openclaw.json` as a template in git.** The docs
   describe editing the live file directly and relying on hot-reload.
   We maintain a scrubbed template in `platform/openclaw.json`
   and write it to `~/.openclaw/openclaw.json` via `setup.sh`. This
   is stricter than the docs prescribe — it means config changes
   must go through `setup.sh` or be applied via the `gateway` tool,
   not just hand-edited. Reasonable for a multi-agent setup, but
   means the Control UI config editor and `openclaw configure` wizard
   are effectively bypassed (any changes they make would be
   overwritten on next `setup.sh` run).

---

### 5.4 MC installation: docs vs. our approach

**What the docs say:**

The canonical install path is the interactive installer:
```bash
curl -fsSL https://raw.githubusercontent.com/abhi1693/openclaw-mission-control/master/install.sh | bash
```
or `./install.sh` from a local clone.

The installer is interactive, asks for deployment mode (Docker or
local), installs missing dependencies, generates `.env`, and starts
the stack. The docs call this "option A: one-command production-style
bootstrap" and describe it as the recommended path.

Alternatively, manual setup: `cp .env.example .env`, set
`LOCAL_AUTH_TOKEN` (minimum 50 characters), then
`docker compose -f compose.yml --env-file .env up -d --build`.

The docs use `compose.yml` (hyphen) consistently. Frontend defaults
to `localhost:3000`, backend to `localhost:8000`. `NEXT_PUBLIC_API_URL`
defaults to `auto` which resolves to the current host — the docs call
this out explicitly as needing an override behind a reverse proxy.

**What we do:**

We don't use the installer at all. `setup.sh` clones the MC repo,
generates credentials, writes `platform/.env`, builds Docker images
from source, and runs `docker compose -f compose.yaml`. This is
closest to the manual setup path, but with several divergences:

1. **We use `compose.yaml` (no hyphen).** The MC docs and repo
   consistently use `compose.yml`. We renamed it in our fork to
   match Docker Compose conventions. This is cosmetic but creates
   drift from upstream — when reading MC docs, the file reference
   won't match.

2. **We build from source on every install.** The MC docs describe
   a standard Compose workflow that uses pre-built images where
   available. We always build from the local `mission-control/`
   clone. This is slower and means our running version can drift
   from upstream without it being obvious.

3. **We added non-standard env vars and extra_hosts.** The `compose.yaml`
   now includes `NEXT_PUBLIC_API_URL` pointing to the Tailscale
   hostname, `extra_hosts` with a hardcoded Tailscale IP, and the
   socat proxy service. None of these are in the upstream MC
   reference config. They're necessary for our deployment topology
   (Tailscale + Docker bridge) but they mean our compose file has
   diverged significantly from upstream and will need manual review
   on every MC upgrade.

4. **We bypass the MC database backup.** The installer's `.env`
   generates `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`,
   and `POSTGRES_PORT` — all referenced in the backup runbook. We
   set `POSTGRES_DB` and `POSTGRES_PASSWORD` but the backup cron
   that the MC docs describe is not wired up. The credentials exist;
   the scheduled dump does not.

5. **The gateway is connected via socat proxy, not directly.** The
   MC docs assume the gateway is reachable at a normal URL. Our
   architecture routes through `172.20.0.1:18790` via a socat
   bridge because Tailscale Serve strips WebSocket query params.
   This works but is a bespoke workaround that upstream MC does not
   know about. It means upstream WS connection code changes could
   break us silently.

**Critical finding — MC auth token management:**

The MC docs describe a token re-sync procedure
(`templates/sync?rotate_tokens=true`) that MC uses to write
`AUTH_TOKEN` into agent workspace files via gateway templates. This
is the upstream-intended provisioning flow: MC is the authority on
agent tokens; it pushes them to the gateway.

We do the opposite: `setup.sh` generates tokens via direct DB
write (PBKDF2) and writes them to each agent's `.env` manually.
MC is not the authority on tokens in our setup — `setup.sh` is.
This works but means the MC-native provisioning and template sync
features don't apply to us. If we ever want to use gateway templates
or agent lifecycle management from the MC UI, we'd need to reconcile
the token authority model.

---

### 5.5 Summary of gaps

| Area | Documented recommendation | Our current state | Risk |
|------|--------------------------|-------------------|------|
| MC state backup | Daily `pg_dump` | Not implemented | High — full task history lost on reinstall |
| OpenClaw backup | `openclaw backup create` | Not implemented | Medium — identity files covered, sessions not |
| MC token authority | MC provisions tokens via templates/sync | setup.sh writes directly to DB | Medium — MC lifecycle features unavailable |
| MC ask-user bridge | Agent → MC → gateway main → Telegram | Unused | Low — plan-mode covers it today |
| `openclaw.json` editing | Live edit + hot-reload or wizard | Template + setup.sh | Low — intentional, but bypasses Control UI |
| `compose.yaml` drift | Minimal upstream config | Significant divergence | Medium — upgrade path needs manual review |
| Old Docker README | N/A (upstream doesn't have this) | Stale docs in /home/pi/openclaw/ | Low — confusing but not operational |

The highest-priority item — by significant margin — is the missing
MC database backup. Everything else is either intentional or low
risk. The DB backup should be added to `setup.sh` and the nightly
cron before any further MC integration work.

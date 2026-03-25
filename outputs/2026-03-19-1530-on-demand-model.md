> **Question:** I want to explore disabling heartbeats altogether and doing everything on demand — via the chat window or a WhatsApp message. So I review PRs and then tell the agent I have an update. Or the programme manager participates in the IDEA WhatsApp group and responds to messages and receives commands via messages. What do you think of this setup? Which features of the proposal doc are not possible with this setup?

## Short answer

This is a sound model for your current stage. The on-demand approach trades a small set of proactive conveniences for zero idle cost and full control over when agents run. The WhatsApp-as-command-interface idea for Marco is genuinely well-suited to his role. The things you lose are either not yet applicable (boards are empty), solvable without LLM sessions, or genuinely low-priority.

---

## What changes in the on-demand model

### What stays fully intact

Everything that is reactive — agents doing work you assign — is completely unaffected:

- You create a task in Mission Control, assign it, open the chat, approve the plan → agent executes. Unchanged.
- You review a PR on GitHub, want quality-manager's view → open QM's chat, paste the PR link. Unchanged.
- You message Marco via WhatsApp or MC chat: "draft a donor update" → Marco proposes a plan → you approve. Unchanged.
- Standups: you open each agent's tab and say "contribute to today's standup". Unchanged.
- Discussion threads, proposals, design docs — all CEO-initiated anyway. Unchanged.

### What you lose

There are four things the proposal relies on proactive scheduling for:

---

**1. BACKLOG.md auto-export**

The proposal has a cron job refreshing `BACKLOG.md` from Mission Control every 2 hours. Without it, `BACKLOG.md` goes stale and agents reading it at session start get outdated task state.

**Verdict: easy to solve without an LLM.** The export script is a pure bash/API call — no intelligence needed. A cron entry running `scripts/export-backlog.sh` and committing the result is a 3-line script job, not an agent wake. Zero token cost. This one should stay as a cron script regardless of the heartbeat decision.

---

**2. Morning standup seeding**

The proposal seeds `standups/YYYY-MM-DD.md` at 07:30 with recent commits, open PRs, and backlog priorities.

**Verdict: minor, and also solvable without an LLM.** The seed content (git log output, open PR list, backlog export) is deterministic — a bash script can generate it. Only if you want the opening section to include synthesised commentary ("three PRs open, the engine one has been blocked 4 days") do you need an LLM. For now, a simple script that dumps the raw facts is sufficient. You can always kick off the standup manually.

---

**3. Proactive deadline and blocker surfacing**

This is the most legitimate thing heartbeats provide: an agent that notices a grant deadline approaching and tells you, without being asked. Or quality-manager flagging that a PR has been waiting 5 days.

**Verdict: real value, but not yet applicable.** There are no grants being tracked, no PRs open, no field reports coming in. This is a future need, not a present one. When the time comes, there are two ways to solve it cheaply:

- **Lightweight cron script** (no LLM): check `grant-tracker.md` for upcoming deadlines, check GitHub API for stale PRs, send a WhatsApp message if thresholds are crossed. This is 20 lines of bash, zero tokens.
- **One daily "digest" agent activation** for Marco specifically: wake once at 08:30, scan for anything time-sensitive, message you on WhatsApp if anything needs attention, otherwise do nothing. Cheap because it's one short activation per day, not 6–144.

---

**4. Overnight autonomous progress**

The proposal's vision: agents making progress while you sleep, so you wake up to work already done. This is the main thing you fully give up with pure on-demand.

**Verdict: not applicable yet, and overstated for this stage.** Autonomous overnight work only makes sense when:
- There is a well-defined backlog with approved tasks
- Agents have enough context to make progress independently
- You trust the outputs enough to review them the next morning rather than approving each step

None of these conditions are met yet. The boards are empty. Even when they aren't, IDEA's plan-mode governance means agents always stop for approval before acting — they can start a plan overnight but can't execute it. Overnight heartbeats on empty boards are pure cost with zero output.

---

## The WhatsApp model for Marco

This is the strongest part of the proposal, and it works especially well in an on-demand model.

**Why it fits Marco's role:**
- Programme management is conversational work. Field coordination, supporter comms, grant research — these are driven by conversation and context, not by polling a task board.
- WhatsApp is already the channel for IDEA's stakeholders. Having Marco there is natural, not forced.
- You're probably already in the IDEA supporters group on your phone. Marco being a contact you can message from the same interface is zero-friction.

**How it would work in practice:**
- You're reading something on your phone and think "I should ask Marco to check that grant" → WhatsApp message → Marco responds in the OpenClaw backend, proposes a plan → you approve → work happens
- A supporter sends a message in the group → Marco can draft a response → you approve before it's sent
- You want a donor update before a meeting → "draft a donor update, keep it short" → Marco delivers a draft

**What it doesn't require:** heartbeats. Marco wakes when you message. That's on-demand, zero idle cost, and the most natural interaction model for that role.

**The one dependency:** WhatsApp setup requires a dedicated SIM and QR pairing (see proposal's WhatsApp section). This hasn't been done yet. Until it is, MC chat is the equivalent.

---

## Revised model: on-demand + two lightweight cron scripts

| Mechanism | What it does | LLM? | Cost |
|-----------|-------------|------|------|
| `export-backlog.sh` cron (2h) | Keeps BACKLOG.md fresh | No | ~Zero |
| `standup-seed.sh` cron (07:30 Mon–Fri) | Creates standup file with raw context | No | ~Zero |
| CEO → MC chat | Assigns tasks, approves plans, gets updates | Yes | On demand only |
| CEO → WhatsApp → Marco | Commands, questions, review requests | Yes | On demand only |
| WhatsApp group → Marco | Incoming supporter/field messages for draft response | Yes | On demand only |
| All heartbeats | Disabled | — | Zero |

The two cron scripts are trivial shell jobs that touch no LLM. Everything else is triggered by you.

---

## What this means for the proposal

The proposal needs one conceptual correction: the "always working, always alive" framing was premature and misapplied. It is the right long-term aspiration for when there is a flowing backlog and field operations are active. It is the wrong setup for a bootstrapping phase with empty boards.

The scheduling table in the proposal (30-minute heartbeats for all agents) should be revised to:
- `export-backlog.sh` cron: keep as-is (no LLM)
- `standup-seed.sh` cron: keep as-is (no LLM, or very cheap)
- All agent heartbeats: disabled until boards have active tasks and you decide autonomous monitoring has earned its cost

The WhatsApp section of the proposal is unchanged — it's already framed as on-demand command/response. Good as written.

---

## Recommendation

Disable all heartbeats now. Write two simple cron scripts (backlog export, standup seed) that require no LLM. Establish the WhatsApp setup when you have the SIM. Revisit heartbeats only when there's a clear, specific monitoring need that can't be handled by a bash script or by you opening a chat tab.

The on-demand model is not a step backward from the proposal. It's the right model for this stage.

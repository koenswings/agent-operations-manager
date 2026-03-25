> **Question:** Read the proposal doc. Why does Marco even need multiple heartbeats per day? Are they not supposed to become active once per day to tackle the next job? And then whenever the CEO asks them something?

## Short answer

You're right, and the proposal got this wrong. The original design prescribed 30-minute heartbeats for all 5 agents uniformly — including Marco — without distinguishing between roles that need rapid feedback loops and roles that don't. For Marco, a once-daily activation plus on-demand response is the correct model.

---

## Why the proposal went with frequent heartbeats

The proposal frames heartbeats as giving the company the feel of a team that is *"always working, not tools waiting to be used"*. It lists what each heartbeat does for Marco:

> "Checks assigned backlog items; surfaces grant deadlines, pending field reports, drafts awaiting CEO review"

The implicit assumption is that these things change frequently enough to justify checking every 30 minutes. They don't. Grant deadlines don't shift hourly. Field reports don't arrive at 30-minute intervals. Draft review isn't time-critical to the minute. The argument that made sense for engine-dev (where PRs open, CI runs finish, test failures appear throughout the day) was copy-pasted onto Marco without asking whether it applies.

The result is paying for 48 wake-ups per day per agent across a 24-hour window (or ~28 across the configured 08:00–22:00 active hours), for a role whose entire work cycle is measured in days and weeks.

---

## What the proposal missed: heartbeat frequency should match feedback loop speed

The right question for each agent is: **how fast can the relevant state change, and what is the cost of acting on stale state?**

| Agent | What it monitors | How fast it changes | Right cadence |
|-------|-----------------|---------------------|---------------|
| engine-dev | PRs, test runs, CI status | Hours — a PR can open any time | Every 1–2h during active hours |
| console-dev | PRs, assigned tasks | Hours | Every 1–2h during active hours |
| quality-manager | New PRs waiting for review | Hours | Every 1–2h during active hours |
| site-dev | PRs, deploy status | Hours | Every 1–2h during active hours |
| programme-manager | Grant deadlines, field reports, comms drafts | Days to weeks | Once daily, or on demand |

Even for the developer agents, 30 minutes was probably too aggressive. 1–2 hours during working hours is more than sufficient — a PR waiting 90 minutes for a review check is fine.

---

## The right model for Marco

Your description — **once per day to pick up and work on the next task, plus on-demand when you ask** — is correct for programme management work. Here's why:

**Marco's tasks are long-cycle by nature.** Writing a grant proposal, drafting a teacher guide, researching funding opportunities, coordinating a school visit — these take hours to days of work per task. Marco doesn't need to "check" anything between tasks. It needs to:
1. Wake up once per day, see what's on the board, pick up the next item, make progress, stop.
2. Respond immediately when you open a chat and ask something.

**On-demand is already free.** When you open Marco's chat in Mission Control and type something, it responds — no heartbeat needed. The heartbeat is only for *unsolicited* proactive behaviour. For Marco, unsolicited behaviour once per day is enough: "anything new on the board? any deadlines approaching? otherwise I'll continue on my current task."

**The cost of idle heartbeats is real.** Even a heartbeat that takes 1 second still:
- Wakes an LLM session
- Loads context (HEARTBEAT.md, MEMORY.md, AGENTS.md, task state)
- Makes API calls to Mission Control to check board state
- Returns HEARTBEAT_OK

That's ~5–15 API calls and a non-trivial number of tokens per heartbeat, multiplied by every agent that's running on a schedule.

---

## Recommended revised schedule

| Activity | Mechanism | Schedule |
|----------|-----------|----------|
| Morning standup seed | Cron | 07:30 Mon–Fri |
| BACKLOG.md export | Cron | Once at 07:00 |
| engine-dev heartbeat | Heartbeat | Every 2h, 08:00–20:00 |
| console-dev heartbeat | Heartbeat | Every 2h, 08:00–20:00 |
| quality-manager heartbeat | Heartbeat | Every 2h, 08:00–20:00 |
| site-dev heartbeat | Heartbeat | Every 2h, 08:00–20:00 |
| programme-manager heartbeat | Heartbeat | Once daily at 08:30 |

This is a conservative starting point. Once tasks are flowing and there's real work on the boards, you can tune cadences based on observed behaviour.

**Until tasks are assigned to boards:** consider disabling all heartbeats entirely. A heartbeat on an empty board produces nothing useful — Marco wakes up, checks, finds nothing, returns HEARTBEAT_OK, goes back to sleep. Zero value, non-zero cost.

---

## What to change

Two things need updating:

**1. `jobs.json` in the OpenClaw data volume** — already updated from 10m to 60m in the previous session. Should go to once-daily (or be disabled until tasks are assigned).

**2. The proposal's "Scheduled activities" table** — should be updated to reflect the revised cadences above, so it stays accurate as the reference document.

**3. HEARTBEAT.md in Marco's workspace** — the line *"Keep cadence conservative unless there is a clear latency need"* is good, but doesn't prevent whoever configured the cron job from picking an aggressive default. The `jobs.json` entry is the ground truth; the HEARTBEAT.md description is advisory.

---

## Practical next step

When you're ready to restart the system:
1. Set Marco's cron job to once daily — or disable it until you assign first tasks
2. Don't create cron jobs for the other 4 agents yet — wait until there's real work on their boards
3. When the boards have tasks, create heartbeat jobs at 2h intervals for the developer/QM agents

The proposal's aspiration — a team that feels alive and is always working — is valid. But it only costs money when agents have something to report on. An empty board checking in every 10 minutes is just a leaking tap.

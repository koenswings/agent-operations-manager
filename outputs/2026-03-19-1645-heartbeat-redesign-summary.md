> **Question:** Update the proposal with the renewed approach. Provide a summary of the heartbeat redesign. Automate the standup agent chain. Update PR review to chat-preferred.

## Heartbeat redesign — summary

### What changed and why

| | Before | After |
|---|---|---|
| **Default state** | All 5 agents heartbeating every 10–30 minutes | All heartbeats disabled |
| **Marco (programme mgr)** | Every 10 minutes (2,385 runs since Mar 2) | On-demand via chat or WhatsApp |
| **Developer agents** | Every 30 minutes, 08:00–22:00 | On-demand; re-enable at 2h cadence when boards have active tasks |
| **Cost when idle** | Tokens consumed on every heartbeat regardless of board state | Zero |
| **Trigger for re-enabling** | N/A | Sustained flow of active tasks that genuinely benefit from monitoring |

**Root cause of the budget leak:** Marco had a cron job running at 10-minute intervals since March 2. No other agents had cron jobs. 2,385 runs × tokens per run = significant idle spend on an empty board.

**Design flaw in original proposal:** the "always alive" framing led to uniform 30-minute heartbeats for all agents without asking whether each role's relevant state changes fast enough to justify it. Programme management work (grants, field reports, comms) changes over days and weeks, not minutes. Developer-agent work (PRs, CI) changes over hours at most.

### When to re-enable heartbeats

Re-enable per agent only when:
1. The agent's board has active tasks
2. There is something the agent can usefully notice between your sessions with it
3. The cost per heartbeat is worth the proactive surfacing it provides

Suggested cadences when re-enabling:
- Developer agents + QM: every 2 hours, 08:00–20:00
- Programme manager: once daily at 09:00 (or stay on-demand — WhatsApp covers this role better)

### What replaces heartbeats

**For all proactive monitoring:** nothing, until there is something worth monitoring. An empty board checked every 2 hours is still a waste.

**For deadline/blocker surfacing (future):** a lightweight cron script (no LLM) that checks `grant-tracker.md` for approaching deadlines and sends a WhatsApp message to the CEO if a threshold is crossed. Zero tokens.

**For daily rhythm:** the automated standup chain (see below) provides a structured morning pulse without any idle polling.

---

## Automated standup chain

### Full schedule

| Time | Job | LLM? | What happens |
|------|-----|------|-------------|
| 07:30 | `standup-seed.sh` | No | Generates context, checks hash, writes standup file or "skipped" |
| 07:35 | Quality Manager | Yes | Writes Summary + QM contribution (skips if standup skipped) |
| 07:45 | engine-dev | Yes | Contributes (skips if standup skipped) |
| 07:55 | console-dev | Yes | Contributes (skips if standup skipped) |
| 08:05 | site-dev | Yes | Contributes (skips if standup skipped) |
| 08:15 | programme-manager | Yes | Contributes (skips if standup skipped) |

CEO arrives to a completed standup. Adds CEO close section. Total CEO time: 5–10 minutes reading.

### Change detection

The seed script hashes: git log across all repos, open PR titles/states, BACKLOG.md content, new files in `proposals/` and `discussions/`. If the hash matches yesterday's, all cron jobs detect the "skipped" marker and do nothing. **No LLM is woken on unchanged days.**

Monday exception: always run regardless of hash.

### Cost per standup

Each agent self-reports token counts in a table at the bottom of the standup file. Rough estimate for a typical standup with 5 agent contributions at ~5,000 input / ~400 output tokens each:

- ~25,000 input tokens + ~2,000 output tokens total
- At Opus 4.6 pricing (~$15/M input, ~$75/M output): ~$0.38 + ~$0.15 = **~$0.53 per standup**
- 5 days/week × 52 weeks = **~$138/year** if run every working day

This is the upper bound — skipped standups cost nothing, and standups where only 2–3 agents have active work cost proportionally less.

### Quality Manager's dual role in standup

QM runs first (07:35) and does two things in one session:
1. Writes the **Summary section**: cross-project synthesis of what the context means — which PRs are stale, what's moving, what needs attention
2. Writes **QM's own contribution**: what QM is working on, insights, questions

Later agents read QM's synthesis as part of their context, which gives the whole standup a shared framing before individual agents add their voice.

---

## PR review — chat-first

### What changed

**Before:** recommended GitHub inline comments + chat notification.

**After:** chat-first. CEO describes changes needed at whatever level of detail is useful — high-level direction ("the approach to error handling is wrong, think about this differently") or specific corrections ("rename that function"). Agent implements, pushes to the same branch, PR updates automatically. CEO merges on GitHub when satisfied.

### Why

- The correction process doesn't need to be documented. What matters is a well-documented final solution — which is in the merged code and its commit history.
- Line-level review is sometimes useful but not always needed. Chat gives flexibility to work at whatever altitude is appropriate.
- The GitHub PR remains the formal record of *what* changed; the chat conversation explains *why* the final version looks the way it does, visible to the CEO in MC.

### Mechanics (unchanged)

The agent always pushes to the existing branch. The PR updates automatically. No new PR is created for the same change. The CEO merges on GitHub when the final state is correct.

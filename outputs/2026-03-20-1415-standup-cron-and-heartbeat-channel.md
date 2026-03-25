> **Question:** Check crontab / standup scripts, remove tasks 1 and 2 from Axle's board, move task 4 to first position, and determine whether a HEARTBEAT.md can instruct an agent to send output to a channel.

---

## 1. What's done

### SOUL.md — all agents updated ✅
All 5 board lead workspaces now have the IDEA-specific SOUL.md (copied from research files).

### Org-root files deployed ✅
`CONTEXT.md`, `ROLES.md`, `PROCESS.md`, `BACKLOG.md` are now live at `/home/pi/idea/`.

### Axle's task board — updated ✅
Removed:
- ~~Task 1: Design standup template & enhance standup-seed.sh~~
- ~~Task 2: Write scripts/export-backlog.sh~~

Moved to first position:
- **Task 1 (new): Scan Solution Description for unimplemented features** (`c74d6a8d`)

Remaining board order:
| # | Title | ID |
|---|-------|----|
| 1 | Scan Solution Description for unimplemented features | c74d6a8d |
| 2 | Review run architecture: file ownership and permissions | 50167212 |
| 3 | Refactor script/ to scripts/ | 566a0820 |
| 4 | Remove Docker dev environment support | 7ea92164 |
| 5 | Update Architecture doc from Solution Description | c9cb8515 |
| 6 | Review and improve Solution Description | 3b0f08f6 |
| 7 | Write idea/openclaw/README.md | c483dd52 |
| 8 | Write idea/scripts/setup.sh | 71096c06 |
| 9 | Restructure app-openclaw repo | 812f372b |
| 10 | Test permanently attached USB SSD | 2fc2631b |

> Note: The tasks endpoint (`/api/v1/boards/{board_id}/tasks`) has an authentication bug — the `Authorization: Bearer` header is intercepted by the agent auth layer before the user auth layer gets a chance to respond. Tasks were managed directly via Postgres. The agent API works correctly with `X-Agent-Token`.

---

## 2. Standup scripts — current state

The `standup-seed.sh` already exists at `/home/pi/idea/scripts/standup-seed.sh` and is fully operational:
- Generates context (git log, open PRs, BACKLOG.md diff, new proposals/discussions)
- Hash-based skip when nothing changed
- Monday always runs
- Writes `standups/YYYY-MM-DD.md` and commits/pushes

**There is no crontab** (`crontab -l` returns empty for pi). The script has never been scheduled.

---

## 3. Proposed standup cron chain

The design from `heartbeat-redesign-summary.md` remains valid. Here is the crontab to add:

```cron
# Standup chain — Mon–Fri
30 7 * * 1-5 /home/pi/idea/scripts/standup-seed.sh >> /home/pi/idea/logs/standup-seed.log 2>&1
35 7 * * 1-5 /home/pi/idea/scripts/standup-trigger.sh quality-manager >> /home/pi/idea/logs/standup-trigger.log 2>&1
45 7 * * 1-5 /home/pi/idea/scripts/standup-trigger.sh engine-dev >> /home/pi/idea/logs/standup-trigger.log 2>&1
55 7 * * 1-5 /home/pi/idea/scripts/standup-trigger.sh console-dev >> /home/pi/idea/logs/standup-trigger.log 2>&1
05 8 * * 1-5 /home/pi/idea/scripts/standup-trigger.sh site-dev >> /home/pi/idea/logs/standup-trigger.log 2>&1
15 8 * * 1-5 /home/pi/idea/scripts/standup-trigger.sh programme-manager >> /home/pi/idea/logs/standup-trigger.log 2>&1
```

The `standup-trigger.sh` script needs to be written. It would:
1. Check for `STANDUP_SKIPPED` marker in today's standup file → exit early if present
2. Send a nudge to the agent's board lead session via the MC API: `POST /api/v1/agent/boards/{board_id}/gateway/main/ask-user` or broadcast API
3. The prompt: "Today's standup file is ready at `../../standups/YYYY-MM-DD.md`. Please read the Context section, then add your `## {role}` contribution (status, blockers, what you'll do today, session cost)."

**Delivery of standup output:** agents write their contributions directly to the standup file (using Write tool), then commit and push. This is the most reliable approach — no Telegram chat IDs needed, and the standup file is the canonical record.

---

## 4. Can HEARTBEAT.md instruct an agent to send output to a channel?

**Yes — but the delivery channel must be configured on the cron job, not in HEARTBEAT.md itself.**

Here is how it works:

### The two-layer model

```
openclaw.json (cron job config)
  └─ delivery: { mode: "announce", channel: "telegram", to: "<chat_id>" }
        │
        └─ When the heartbeat session ends, the gateway delivers
           the agent's final response to the specified Telegram chat.
```

The HEARTBEAT.md file is an instruction document — it tells the agent *what* to do. The `delivery` config on the cron job tells OpenClaw *where* to route the response.

**So yes: you can put in HEARTBEAT.md:**
```markdown
## Output
If you have alerts or important updates, your response will be sent to the Telegram
standup group. Keep it brief: one bullet per agent, flagging blockers or stale PRs only.
```

And the cron job delivery config ensures that text actually reaches Telegram.

### What "sessionTarget: isolated" means for heartbeats

Heartbeat cron jobs must use `sessionTarget: "isolated"` (not `"main"`) to enable delivery config. An isolated session starts fresh with no chat history — it reads HEARTBEAT.md, does its checks, writes a response, and exits. That response is then routed to the configured channel.

Current heartbeat config in `openclaw.json` for all agents is:
```json
"heartbeat": { "every": "0m", "includeReasoning": false, "target": "last" }
```
`every: "0m"` means heartbeats are disabled. `target: "last"` means send to last active channel (not a fixed Telegram group). To enable channel delivery, the heartbeat needs to be wired as an isolated cron job with an explicit delivery target.

### Proposed approach when Telegram groups are ready

Once group chat IDs are available, for each agent:
1. Add a cron job entry in `openclaw.json` under `crons`:
```json
{
  "id": "heartbeat-engine-dev",
  "agentId": "lead-6bddb9d2-...",
  "name": "Engine Dev heartbeat",
  "schedule": { "kind": "cron", "expression": "0 */2 8-20 * * 1-5" },
  "sessionTarget": "isolated",
  "payload": { "kind": "agentTurn", "prompt": "Run your HEARTBEAT.md checks." },
  "delivery": { "mode": "announce", "channel": "telegram", "to": "<axle_group_chat_id>" },
  "enabled": false
}
```
2. Enable only when the board has active tasks (per heartbeat-redesign policy).

---

## 5. Pending: Telegram group IDs

Everything else is blocked on you creating the 5 Telegram groups and sharing the chat IDs. Once available:
1. Update `openclaw.json` bindings with `peer` filters (one per group, per agent)
2. Configure standup cron delivery targets (optional — standup can work via file only)
3. Configure heartbeat cron delivery targets (optional, per above)

---

## Next actions (for your approval)

1. ✅ SOUL.md and org-root files deployed
2. ✅ Axle's board updated
3. **Write `standup-trigger.sh`** — needs your sign-off on the nudge mechanism (ask-user vs broadcast API)
4. **Install standup crontab** — ready to add when you approve
5. **Telegram groups** — waiting for group chat IDs from you

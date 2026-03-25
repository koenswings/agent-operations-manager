> **Question:** Please analyse the current telegram interactions and spot any anomalies — I already see that agents are not contributing to the standup and I also do not see any proposals from Axle on his current task. Is there also a way to send direct messages to each individual agent (eg using /axle)? Also: I see that Axle is responding by default — why is that and why are the other agents silent.

---

## Diagnosis: Telegram Routing Anomalies

### 1. Why Axle responds and all other agents are silent

**Root cause: first-match wins in the bindings array.**

The routing engine (`resolve-route.ts`) evaluates all 5 bindings against the incoming Telegram message. All 5 have identical match criteria:

```json
{ "match": { "channel": "telegram" } }
```

No `peer`, `accountId`, `guildId`, or `teamId` constraint distinguishes any of them. The router runs through priority tiers (peer → guild+roles → guild → team → account → channel) and all 5 bindings tie at the "account" tier (because `accountPattern = ""` which is `!== "*"`). The router calls `.find()` — **first match wins** — and Axle (lead-6bddb9d2) is listed first in the array.

All other agents never receive any Telegram message.

---

### 2. Why agents are not contributing to standups

**The agents have no mechanism to proactively push to Telegram.**

The heartbeat design (HEARTBEAT.md) defines what agents check during a heartbeat poll, but the output goes into memory files, not to any channel. Agents only send Telegram messages in two situations:
- **Reactive**: they receive an inbound Telegram message (which, per issue 1, only Axle receives)
- **Proactive cron**: a cron job is configured to trigger the agent and send output to "last" channel — but this requires a prior session to know _where_ to send, and no standup cron has been configured

The standup automation described in PROCESS.md has never been wired to Telegram. This is a design gap: we designed heartbeat content but not the delivery path.

---

### 3. Why there are no proposals from Axle on task 1

Two compounding problems:

**a) Task is still in `inbox`, not `in_progress`**
Axle's MEMORY.md (updated 2026-03-20 04:03 UTC) shows his directive was received and his "next step" is to pick up task 1 (`1bba406c` — Design standup template). He has noted it but not yet acted on it — the session likely ended before he could start. He is unblocked, just waiting for the next heartbeat poll.

**b) Axle's SOUL.md is the generic OpenClaw default, not IDEA's custom one**
The workspace file at `workspace-lead-6bddb9d2-.../SOUL.md` is the generic "lead agent for this board" soul that OpenClaw initialises by default. It was overwritten during provisioning (as warned in memory). The custom `SOUL.md` from `research/openclaw-initial-config/agent-engine-dev/SOUL.md` was never re-applied. Axle is therefore missing his identity as IDEA's Engine Developer and does not know his specific role, constraints, or working style.

Additionally, the **org-root files** (CONTEXT.md, ROLES.md, PROCESS.md, BACKLOG.md) have not been deployed to the actual `/home/pi/idea/` root — Axle is working in a contextual vacuum.

---

### 4. Can you send direct messages to individual agents?

**Not with the current setup. The routing engine has no text-prefix dispatch.**

The binding `match` object supports: `channel`, `accountId`, `peer { kind, id }`, `guildId`, `teamId`, `roles`. There is no `textPrefix` or `command` field — `/axle some message` will not be parsed and routed to Axle specifically. The `native: auto` commands config is for OpenClaw's own built-in slash commands, not user-defined agent routing.

**Two viable approaches:**

**Option A — Separate Telegram groups, one per agent (recommended)**
Create 5 Telegram groups (e.g. "IDEA · Axle", "IDEA · Pixel", etc.), add the bot to each, and bind each group to one agent using a `peer` filter:

```json
{
  "agentId": "lead-6bddb9d2-...",
  "match": {
    "channel": "telegram",
    "peer": { "kind": "group", "id": "<axle_group_chat_id>" }
  }
}
```

Peer bindings are the highest-priority tier — they will win over any channel-only binding. This is also the pattern that naturally scales: you message the group for the agent you want, and the current catch-all binding (first in array) handles any messages from the main/general chat.

**Option B — Fix the broadcast: one binding per agent, differentiated by accountId**
If all agents should all see every message (broadcast), the current design almost achieves this, but the routing prevents it. OpenClaw does have a broadcast API (`/api/v1/gateway/lead/broadcast`) that MC uses — but this is a server push, not a Telegram-initiated route. Bindings are single-dispatch only; you cannot fan-out one Telegram message to all 5 agents via bindings alone.

---

## Summary of Issues and Proposed Fixes

| # | Issue | Severity | Fix |
|---|-------|----------|-----|
| 1 | All Telegram messages route to Axle only | High | Add `peer` filter to each binding (Option A), or restructure with one dedicated group per agent |
| 2 | No standup delivery path | Medium | Add a heartbeat cron that fires a standup prompt at each agent via MC's broadcast API or cron (separate from Telegram routing) |
| 3 | Axle's SOUL.md is generic, not IDEA-specific | High | Re-copy `research/openclaw-initial-config/agent-engine-dev/SOUL.md` with `sudo cp` |
| 4 | Org-root files not deployed to `/home/pi/idea/` | High | Deploy `research/openclaw-initial-config/idea/` files (Step 17 from build log) |
| 5 | Axle's task 1 not yet started | Low | Will self-resolve at next heartbeat; but may need a nudge via Telegram or MC |
| 6 | MC has no `restart: unless-stopped` | Medium | Add restart policy to `compose.yml` (known pending item) |

---

## Recommendation order

1. **Fix SOUL.md and org-root deployment first** — Axle cannot produce meaningful proposals without knowing who he is and what IDEA is doing
2. **Fix the Telegram routing** — decide: separate groups (Option A) or accept that Telegram is CEO → Axle only, and use MC UI for other agents
3. **Wire standup delivery** — design a cron job that sends a standup prompt to each agent and routes their response back to a summary channel or file

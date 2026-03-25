> **Question:** What is the added value of OpenClaw and Mission Control to this project? Which features are we not using?

*Based on reading all local MC/OpenClaw documentation, the full gateway API spec, and querying the live MC instance.*

---

## Live state snapshot (queried 2026-03-22)

| Item | State |
|------|-------|
| Boards | 5 boards, all configured, all **0 tasks** |
| Board groups | 2 groups: **Engineering** (Engine, Console, Site, Quality) and **HQ** (Programme Mgr) |
| Agents | All 5 online in MC, all currently **offline** (last seen: 2026-03-19) |
| Heartbeat schedule | `every: 0m` on all agents — **disabled** |
| Approval gates | `require_approval_for_done: true` on all boards — configured correctly |
| Webhooks / custom fields / tags | **404** — not available in this version of MC |

The task count dropping to zero since the March sessions is notable — those 12 inbox tasks Axle had in March are gone. Either cleared manually or the MC volume was rebuilt.

---

## OpenClaw — what it contributes

OpenClaw is the **runtime layer**. Without it, none of the agent automation exists. Every other capability in this document depends on it.

### What it does that would otherwise require custom build

| Capability | What it means in practice |
|------------|--------------------------|
| **Telegram → Claude bridge** | A message in an agent's Telegram group triggers a Claude session in that agent's workspace. Without OpenClaw this is ~200 lines of custom webhook handling code plus session management. |
| **Plan mode enforcement** | `DEFAULT_PERMISSION_MODE=plan` in compose.yaml enforces that every agent proposes before acting, across all sessions, without relying on agent instructions. A single config line does what would otherwise require per-session CLI flags and trust that agents honour them. |
| **Context pruning** | `cache-ttl` pruning + `softTrim`/`hardClear` prevents long sessions from blowing the context window. Operational agents running multi-step tasks need this — without it, a session doing 12 tasks would context-overflow. |
| **Compaction with memory flush** | Before the context window fills, OpenClaw triggers a memory write (`memory/YYYY-MM-DD.md`) so durable context is saved. Without this, context loss on long sessions means the agent forgets mid-task. |
| **Multi-agent session management** | 5+ agents each in their own workspace, each independently invocable via Telegram or MC, all on the same Pi. OpenClaw manages the concurrent subprocess lifecycle. |
| **Gateway token auth** | Every agent call is authenticated. This is what separates "agent API calls" from "anyone on the local network calling the API". |

### What OpenClaw features are unused

| Feature | Status | Notes |
|---------|--------|-------|
| **Heartbeat scheduling** | Disabled (`every: 0m`) | The mechanism exists to wake agents on a schedule (e.g. every 10 minutes to check their board). Currently disabled — agents only run when you message them. |
| **Skills marketplace** | Unused | Extensible tool packs. Could add capabilities (e.g. browser, database) without modifying agent config. No current need identified. |
| **Subagent concurrency** | Likely unused | `maxConcurrent: 5` for subagents. Would be used if agents spawn sub-agents for parallel work. Not in the current operational model. |
| **QMD memory backend** | Not configured | A semantic memory search system. The baseline config references it but it is not in the live IDEA openclaw.json. Would allow agents to query memory by topic rather than reading files by date. |
| **Boot-md hook** | Unknown | Runs a `BOOT.md` checklist on startup. Not in current agent workspace files. |
| **Command logger hook** | Unknown | Audit log of CLI commands. Potentially useful for debugging but not actively monitored. |
| **Native Telegram command registration** | Auto | Slash commands registered natively in Telegram (e.g. `/standup`). Set to `auto` — may or may not be active. |
| **Board group snapshot** | Unused | Single API call across all 5 agents' boards. Not being checked. |

---

## Mission Control — what it contributes

MC is the **governance and visibility layer**. OpenClaw could run without MC — you'd lose the dashboard, boards, and approval gates, but agents could still be invoked via Telegram or terminal.

### What it does that adds genuine value

| Capability | What it means in practice |
|------------|--------------------------|
| **Board groups** | Two groups are configured: Engineering (4 agents) and HQ (1 agent). This is not just cosmetic — group memory is shared within a group, enabling cross-agent broadcast without direct agent-to-agent messaging. |
| **Goal-typed boards** | Each board has an `objective` and `success_metrics` set during onboarding. When tasks are running, the agent has its goal in the board context, not just in a file. |
| **Approval gate on task closure** | `require_approval_for_done: true` on every board. When this is active, no task can be marked done without CEO approval in MC. This is the key governance control. |
| **CEO dashboard** | All 5 boards visible at once. Agent last-seen timestamps. When tasks are flowing, this is the operational control surface — you see what every agent is doing without navigating 5 directories. |
| **Onboarding workflows** | Used to provision the 5 board leads (Step 15). The goal and success metrics stored in each board came from this. Not a day-to-day feature, but it bootstrapped the correct board configuration. |
| **Activity timeline** | Audit log of all system events. When things go wrong (agent stuck, API error, unexpected behaviour), this is the first place to look. Currently returning 401 from the operator token — needs investigation. |
| **Cross-agent task creation** | Agent A creates a task on Agent B's board via API. This is the mechanism for the auto-review protocol: Axle finishes work, creates a review task on Veri's board, cron triggers Veri. Without MC boards this requires either manual handoff or a custom cross-agent message bus. |

### What MC features are unused or not working in this version

| Feature | Status | Notes |
|---------|--------|-------|
| **Tasks** | Currently 0 on all boards | The core feature of MC is unused. Boards are configured but empty. |
| **Webhooks** | 404 — not in this version | Would allow GitHub CI → MC task creation. A significant workflow bridge that doesn't exist yet in the installed version. |
| **Custom fields** | 404 — not in this version | Org-wide additional metadata on tasks. |
| **Tags** | 404 — not in this version | Task categorisation (the `auto-review` tag used in the HEARTBEAT.md protocol depends on this). Currently broken. |
| **Metrics / dashboards** | Empty | Velocity, cycle-time, WIP charts. Will be empty until tasks are flowing. |
| **Group memory** | 401 from operator token | Was used once (2026-03-20 broadcast directive). Needs correct auth headers to use going forward. |
| **Activity timeline** | 401 from operator token | Auth issue — should be visible to the operator. Needs investigation. |
| **Specialist agents** | `max_agents: 1` on all boards | MC supports lead agents creating sub-specialist agents for parallel work. Not used. Correct for now given the small team model. |
| **SOUL authoring via MC** | Unused | `PUT /api/v1/agent/boards/{id}/agents/{id}/soul` lets you edit an agent's SOUL from within MC. Currently done by editing files directly. |
| **Nudge / broadcast** | Unused | Direct agent-to-agent or CEO-to-all-agents messaging via MC API. Currently handled by CEO manually starting sessions. |
| **HQ board group** | Only Marco | Programme Manager is in HQ alone. A board group of one has no cross-board group memory value. Either other agents should join HQ, or this group structure should be reconsidered. |

---

## The critical gap: tags returning 404

The `auto-review` tag is central to the cross-agent review protocol. An agent creates a review task tagged `auto-review`; a Pi cron detects this tag and triggers the reviewer. If the tags API is returning 404, this entire protocol is broken in the current MC version.

This should be verified before relying on the auto-review workflow.

---

## Summary: what are you actually getting from each system

### OpenClaw — high value, earning its keep

The runtime infrastructure (Telegram bridge, plan mode, context management, session lifecycle) is what makes the agent system work at all. The overhead of running OpenClaw is the Docker container — it is always-on and low-footprint. The unused features (heartbeat scheduling, QMD memory, skills) are genuinely useful for a more autonomous system and represent a natural growth path.

**Verdict: keep, essential.**

### Mission Control — conditional value

MC is currently contributing: board structure (goals, groups, approval rules), the onboarding that set those up, and the dashboard UI for CEO visibility. It is **not yet contributing** the feature it was built for — task flow with approval gates — because no tasks are running.

The 404s on webhooks, tags, and custom fields are concerning: these are in the API spec and documented, but absent from the installed version. Tags in particular matter because the auto-review protocol depends on them.

**Verdict: keep, but with two caveats:**
1. The MC compose.yml needs `restart: unless-stopped` before it can be relied on (Pi reboot = MC disappears)
2. The tag/webhook gaps need investigation — if the auto-review protocol is broken, the main benefit of cross-agent coordination via MC is unavailable

### The honest strategic picture

Both systems are the right choice for a multi-agent team. The complexity you have been feeling is not caused by wrong tool selection — it is caused by the system being in a partially-built state:

- Workspace dirs not yet consolidated to code repos
- MC stack not auto-restarting
- Identity files not committed to code repos
- No tasks flowing through the boards
- Some API features returning 404

Fix those five things and the system delivers what it was designed to do. The architecture is sound. The implementation is incomplete.

---

*Compass · Strategic Advisor · IDEA*

> **Question:** Can you ensure that all the memory of the agents (both Claude memory as well as openclaw memory / identity files) are fully in line with the current virtual company design (see virtual-company-design.md) and especially the changes documented in the operational-flow-design doc

## Agent File Audit — Complete

All 5 operational agent workspace files have been audited and updated to align with `virtual-company-design.md` and `outputs/2026-03-20-1530-operational-flow-design.md`.

---

### What was checked

| File | Source of truth |
|------|----------------|
| `SOUL.md` | Operational flow design — Model B work cycle, auto-review hard rule, heartbeat definition |
| `AGENTS.md` | Virtual company design — agent roles, tech scope; operational flow — cross-agent review protocol |
| `HEARTBEAT.md` | Operational flow design — heartbeat redefinition (external events only, not status checks) |
| `MEMORY.md` | Already updated in previous session — Telegram group names and chat IDs |

---

### Changes applied (all 5 workspaces)

#### SOUL.md — `## How Work Flows` appended

Added to all 5 workspace SOUL.md files + the research SOUL.md:

```
## How Work Flows

Every work cycle begins with a CEO message — nothing moves autonomously.
You produce one of four outputs: PR (code), design doc, proposal, or report.
Cross-agent review: you create an `auto-review` MC task targeting the reviewer agent.
The pi cron picks it up and triggers that agent automatically.

**Hard rule — if this session was triggered by an `auto-review` task:**
Read the artifact. Write your response. Mark the task done. Stop.
Do not create any tasks. Do not continue into other work. No exceptions.

**Heartbeats** — not status checks. A heartbeat monitors one external event source
(CI failures, grant deadlines, stale PRs). If triggered: post a brief Telegram alert
to the CEO. Do not start work. Do not create tasks.
```

#### AGENTS.md — `## Your Role` + `## Cross-Agent Review Protocol` added

Each agent's AGENTS.md now opens with a role-specific `## Your Role` section:

| Agent | Role section covers |
|-------|-------------------|
| Axle | Engine Dev — TypeScript/Node.js/Automerge/Docker; workspace `agents/agent-engine-dev/`; default reviewer Veri |
| Pixel | Console Dev — Solid.js/Chrome Extension; workspace `agents/agent-console-dev/`; default reviewer Veri |
| Beacon | Site Dev — public website; workspace `agents/agent-site-dev/`; default reviewer Veri |
| Veri | Quality Manager — reviews PRs/design docs/proposals across all repos; does not write code; auto-triggered reviewer |
| Marco | Programme Manager — field coordination, teacher guides, supporter comms, fundraising; all outputs are drafts for CEO |

All 5 AGENTS.md files now end with `## Cross-Agent Review Protocol` — the MC API POST template for creating `auto-review` tasks, the full board ID table, and a reference to the auto-review hard rule in SOUL.md.

#### HEARTBEAT.md — `## External Event Monitoring (Future Heartbeats)` appended

All 5 HEARTBEAT.md files now clarify:
- The current file = board startup pre-flight (API connectivity check only)
- OpenClaw heartbeat mechanism is disabled (`every: 0m`)
- Future external event monitors are separate cron scripts per agent:
  - Axle: `heartbeat-ci.sh` — GitHub Actions CI failures
  - Veri: `heartbeat-prs.sh` — stale PRs across all repos
  - Marco: `heartbeat-grants.sh` — approaching grant deadlines
- Alert = brief Telegram message to CEO; no tasks created; no work started

---

### Verification

All changes confirmed present (grep count = 1 per file):
- `"How Work Flows"` — all 5 SOUL.md ✓
- `"Your Role"` — all 5 AGENTS.md ✓
- `"Cross-Agent Review"` — all 5 AGENTS.md ✓
- `"External Event Monitoring"` — all 5 HEARTBEAT.md ✓

---

### What was NOT changed

| File | Reason |
|------|--------|
| `MEMORY.md` | Already updated (Telegram group names + chat IDs) — in line |
| `IDENTITY.md` | Persona/name/pronouns only — no operational model content needed |
| `USER.md` | CEO profile — accurate and complete |
| `TOOLS.md` | Tool manifest — unchanged by operational flow updates |
| `BOOTSTRAP.md` | First-run instructions — no operational flow content needed |

---

### Gaps not yet addressed (require build work, out of Compass scope)

| Gap | Who should build it |
|-----|-------------------|
| `check-new-tasks.sh` — pi cron to auto-trigger reviewer agents | Assign to operational agent (Marco or Axle) |
| `/standup` — CEO-triggered on-demand standup | Assign to operational agent |
| `heartbeat-ci.sh`, `heartbeat-prs.sh`, `heartbeat-grants.sh` | Assign to Axle, Veri, Marco respectively |
| MC `restart: unless-stopped` policy | Infrastructure task — assign to Axle |
| Deploy org-root files (CONTEXT.md, ROLES.md, PROCESS.md, BACKLOG.md) | CEO action or assign to Marco |

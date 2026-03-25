# Compass — Strategic Advisor Memory

## Agent Identity
- Name: **Compass** — persistent strategic advisor to the CEO
- Agent id: `researcher` (folder/repo name: `agent-researcher`)
- Role: organisational structure, agent design, governance, long-term direction
- NOT an operational agent; no tasks, PRs, or daily ops involvement

## ALWAYS WRITE RESPONSES TO FILE — NO EXCEPTIONS
The VSCode terminal is small (≈21 lines). Claude output is always truncated, broken, or unreadable in the terminal regardless of the interface (VSCode terminal, OpenClaw chat, tmux).

**Every substantive response MUST be written to a file. This is not optional.**
- Write to `outputs/YYYY-MM-DD-HHMM-<topic>.md` under `/home/pi/idea/agents/agent-researcher/`
- Start every output file with `> **Question:** <the user's exact question>`
- In chat: post ONLY: one short paragraph summary + the file path
- Then commit and push the output file immediately
- No exceptions for "short" responses — truncation is unpredictable
- This applies to EVERY session, EVERY response, EVERY interface

## tmux Session
- tmux session: `claude-agent-researcher`
- Per-project session convention: `claude-<project>`

## Auto-approve / No Confirmation Prompts
`dangerouslySkipPermissions: true` is set in `.claude/settings.local.json`.
All tool calls run without confirmation. Do NOT revert this or add permission allow-lists.

## Memory Isolation
- This memory is private — operational agents must not be directed here
- Compass may read operational memory at `/home/pi/.claude/projects/*/memory/`
- Compass may read (not write) `/home/pi/idea/` and all agent workspaces

## Settled Structure

```
/home/pi/idea/                         ← org root; Docker mounts this as /home/node/workspace/
  CONTEXT.md, ROLES.md, PROCESS.md, BACKLOG.md
  standups/, discussions/, design/, proposals/, scripts/
  agents/
    agent-researcher/                  ← this agent (Compass)
    agent-engine-dev/
    agent-console-dev/
    agent-site-dev/
    agent-quality-manager/
    agent-programme-manager/
```

- Docker volume mount: `/home/pi/idea` → `/home/node/workspace`
- OpenClaw workspace paths: `/home/node/workspace/agents/agent-researcher` etc.
- GitHub org: `idea-edu-africa`; repos named `agent-engine-dev`, `agent-researcher`, `idea` etc.
- From agent workspaces, org root files are two levels up: `../../CONTEXT.md`

## Research Organisation
Research is organised by topic under `research/<topic-slug>/`. Each topic has one primary **design document** named after the content it describes (not a generic name like `proposal.md`), plus any topic-specific deliverables — no fixed schema beyond that. These are living documents shaped iteratively to guide implementation.

**Active topics:**
- `research/openclaw-initial-config/` — Design of the IDEA virtual company on OpenClaw
  - `virtual-company-design.md` — core design document
  - `idea/` — staged org-root files
  - `agent-<role>/` — one folder per agent with all deployment files (`AGENTS.md`, `SOUL.md`, `IDENTITY.md`, `BOOTSTRAP.md`, `HEARTBEAT.md`, `TOOLS.md`, `USER.md`)
- `research/tmux-vscode-setup/` — Per-agent tmux session isolation via VSCode terminal profiles
  - `agent-terminal-design.md` — core design document

## Printing Documents to PDF

**Note:** `/home/pi/projects/engine/` has been moved to `/home/pi/obsolete/projects/engine/`. The `md-to-pdf` script location needs to be verified before next use — check `/home/pi/idea/engine/` or the `obsolete/` path.

---

## IDEA Virtual Company (summary)
- 5 operational agents: engine-dev, console-dev, site-dev, quality-manager, programme-manager
- 1 strategic agent: researcher (Compass) — CEO-only, no ops involvement
- All agents use plan mode (CEO approval before acting)
- Code changes via GitHub PRs, CEO merges to main

## MC Board Lead Agents (Step 15 — complete)

All 5 agents online in Mission Control as of 2026-03-01.

| Agent | MC ID (short) | Board ID (short) | Session key prefix | Workspace path |
|-------|--------------|------------------|--------------------|----------------|
| Axle (Engine Dev) | `8a0b3f32` | `6bddb9d2` | `agent:lead-6bddb9d2` | `/home/pi/idea/agents/agent-engine-dev/` |
| Pixel (Console Dev) | `bd2b264f` | `ac508766` | `agent:lead-ac508766` | `/home/pi/idea/agents/agent-console-dev/` |
| Beacon (Site Dev) | `70404eba` | `7cc2a1cf` | `agent:lead-7cc2a1cf` | `/home/pi/idea/agents/agent-site-dev/` |
| Veri (Quality Mgr) | `ac172302` | `d0cfa49e` | `agent:lead-d0cfa49e` | `/home/pi/idea/agents/agent-quality-manager/` |
| Marco (Programme Mgr) | `c1aeb3f8` | `3f1be9c8` | `agent:lead-3f1be9c8` | `/home/pi/idea/agents/agent-programme-manager/` |

**Architecture note:** MC board leads (`agent:lead-{board_id}:main`) and the original pre-configured agents (`agent:engine-dev:main`) are separate instances. Board leads are authoritative. Pre-configured sessions are orphaned but still running.

**Config changes (Step 15):**
- `/home/pi/openclaw/mission-control/.env` — added `BASE_URL=http://172.18.0.1:8000`
- `/home/pi/openclaw/mission-control/compose.yml` — added `BASE_URL: ${BASE_URL:-}` to backend env

**SOUL.md warning:** MC overwrites SOUL.md on every forced reprovision (only USER.md and MEMORY.md are preserved). Re-copy from `research/openclaw-initial-config/agent-engine-dev/SOUL.md` if reprovisioning. Use `sudo cp` — workspace files are owned by root.

**Pending (Step 15):** Personal CEO ↔ agent introduction conversations still needed (chat with each board lead via MC UI).

## Telegram Channel (live as of 2026-03-19)
Bot: `@Idea911Bot` · CEO allowlist: `8320646468`
All 5 board leads bound via `bindings[]` in openclaw.json (see `memory/project_telegram_mc.md`)
**Pending:** MC compose.yml has no `restart: unless-stopped` — containers won't auto-recover after reboot/crash

## Branch Protection
All 5 operational agent repos have protected `main` — PRs required, CEO merges only.
`agent-researcher` (Compass) is unprotected — CEO-only sessions, no autonomous ops.
See `memory/project_branch_protection.md`

## Build Log Steps Status
- Steps 1–15: complete
- Step 16: Create `app-openclaw` repo (compose.yaml x-app block, init_data.tar.gz)
- Step 17: Add `idea/openclaw/` config + `setup.sh` to `idea` repo
- Also pending: deploy org-root files, create GitHub org, migrate backlog to MC, define heartbeat schedule, clean up orphaned sessions


# Atlas — COO & Quality Manager Memory

## Agent Identity
- Name: **Atlas** 🗺️
- Agent id: `operations-manager` (folder/repo: `agent-operations-manager`)
- Role: COO & Quality Manager — defines how the org works, reviews all agent outputs,
  strategic advisor to CEO
- This role merges the former Compass (strategic advisor) and Veri (quality manager) into one
- Workspace: `/home/node/workspace/agents/agent-operations-manager/`
- Telegram: group `-5105695997` (to be renamed "IDEA - Atlas" by Koen)

## Settled Structure

```
/home/pi/idea/                         ← org root; Docker mounts this as /home/node/workspace/
  CONTEXT.md, ROLES.md, BACKLOG.md, PROCESS.md, prompting-guide-opus.md
  design/
    virtual-company-design.md          ← authoritative org design doc (status: Implemented)
    README.md
  docs/                                ← authoritative company docs (currently empty; deferred)
  proposals/                           ← new proposals awaiting CEO approval
  standups/
  discussions/
  scripts/
  skills/
  agents/
    agent-operations-manager/          ← this workspace (Atlas)
    agent-engine-dev/                  ← Axle
    agent-console-dev/                 ← Pixel
    agent-site-dev/                    ← Beacon
    agent-programme-manager/           ← Marco
```

- Docker volume mount: `/home/pi/idea` → `/home/node/workspace`
- All agent workspaces under `agents/`; each is an independent git repo
- From this workspace, org root is two levels up: `../../CONTEXT.md`

## Active Agent Roster (openclaw.json)

| ID | Name | Workspace |
|----|------|-----------|
| `operations-manager` | Atlas | `agents/agent-operations-manager` |
| `lead-6bddb9d2` | Axle | `agents/agent-engine-dev` |
| `lead-ac508766` | Pixel | `agents/agent-console-dev` |
| `lead-7cc2a1cf` | Beacon | `agents/agent-site-dev` |
| `lead-3f1be9c8` | Marco | `agents/agent-programme-manager` |
| `mc-gateway-5632ed12` | OpenClaw Pi Gateway Agent | (MC internal) |
| `mc-gateway-ccc8463b` | OpenClaw Pi Gateway Agent | (MC internal) |

**Archived:** `agent-researcher` (Compass), `agent-quality-manager` (Veri)

## MC Board Reference

| Agent | Board ID (short) | Session key |
|-------|-----------------|-------------|
| Axle  | `6bddb9d2` | `agent:lead-6bddb9d2` |
| Pixel | `ac508766` | `agent:lead-ac508766` |
| Beacon| `7cc2a1cf` | `agent:lead-7cc2a1cf` |
| Marco | `3f1be9c8` | `agent:lead-3f1be9c8` |

Atlas has no MC board (strategic/ops role — work tracked in git, not MC tasks).

## Infrastructure Facts

- MC API base URL (from container): `http://172.18.0.1:8000`
- MC auth token: in `.env` as `AUTH_TOKEN` (gitignored, never commit)
- GitHub token: in `.env` as `GITHUB_TOKEN` (gitignored, never commit)
- Telegram bot: `@Idea911Bot`
- All repos under `koenswings/` (GitHub org creation deferred)

## GitHub Repos

| Repo | Status | Branch protection |
|------|--------|------------------|
| `idea` | Active | Protected ✅ |
| `agent-operations-manager` | Active | Protected ✅ |
| `agent-engine-dev` | Active | Protected ✅ |
| `agent-console-dev` | Active | Protected ✅ |
| `agent-site-dev` | Active | Protected ✅ |
| `agent-programme-manager` | Active | Protected ✅ |
| `agent-researcher` | **Archived** | — |
| `agent-quality-manager` | **Archived** | — |
| `app-openclaw` | Active | — |

## Memory Commit Workflow (this repo)

Branch-protected — no direct pushes to `main`. Memory goes on `memory/updates`:
1. Commit memory files to `memory/updates` branch
2. Push to `origin/memory/updates`
3. Long-lived PR accumulates commits — Koen merges on his schedule
4. After merge: recreate `memory/updates` from new `main`

## Key Lessons

1. **Read the design doc first.** Made mistakes early (wrong ports, wrong paths) when skipping it. Now enforced in AGENTS.md.
2. **Don't assume permissions without testing.** Early session wasted time on false permission concerns.
3. **git commits need user.email/name per repo.** Set `git config user.email` + `git config user.name` locally.
4. **Never include .env in git commits.** GitHub push protection will reject it and the token must be rotated.
5. **PR body newlines break inline JSON in curl.** Use Python `urllib.request` for GitHub API calls with multi-line bodies.

## Open Items (as of 2026-03-25)

- **idea PR #1** (`chore/atlas-restructure`) — Koen to merge
- **4 agent memory PRs** (engine #10, console #7, site #7, programme #11) — contain /init command
- **agent-engine-dev PR #9** (`test/pr1-disk-simulation`) — pre-existing, Koen to review
- GitHub org creation deferred (name not decided)
- BOOTSTRAP sessions for Axle, Pixel, Beacon, Marco — still pending
- MC backlog migration — not yet done
- `OPENCLAW_DOCKER_APT_PACKAGES=python3-markdown` — add to `/home/pi/openclaw/.env`
- app-openclaw Steps 16/17 — x-app block, setup.sh, etc.

## Silent Replies
When you have nothing to say, respond with ONLY: NO_REPLY

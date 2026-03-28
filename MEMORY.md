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

| Agent | Board name | Board ID (short) | Session key |
|-------|-----------|-----------------|-------------|
| Atlas | Operations Manager | `d0cfa49e` | `agent:operations-manager` |
| Axle  | Engine Dev | `6bddb9d2` | `agent:lead-6bddb9d2` |
| Pixel | Console Dev | `ac508766` | `agent:lead-ac508766` |
| Beacon| Site Dev | `7cc2a1cf` | `agent:lead-7cc2a1cf` |
| Marco | Programme Manager | `3f1be9c8` | `agent:lead-3f1be9c8` |

## Infrastructure Facts

- MC API base URL (from container): `http://mission-control-backend:8000` (Docker service name — preferred); `http://172.18.0.1:8000` (host bridge — still works, port published)
- MC frontend: `https://openclaw-pi.tail2d60.ts.net:4000`
- MC auth: `AUTH_MODE=local` with `MC_LOCAL_AUTH_TOKEN` in `platform/.env`. Each agent's `.env` stores a **per-agent token** as `AUTH_TOKEN` (not the shared platform token). Per-agent tokens authenticate to `/api/v1/agent/boards/{board_id}/tasks`. Atlas's board (Operations Manager, formerly "Quality Manager") had a new token generated 2026-03-27; agent_token_hash updated in DB.
- Platform: unified compose at `/home/pi/idea/platform/compose.yaml`; 6 services on `idea-net`; migration completed 2026-03-27
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
| `agent-researcher` | **Archived + off-limits** — folder moved to `/home/pi/obsolete/` | — |
| `agent-quality-manager` | **Archived** | — |
| `app-openclaw` | Active | — |

> ⚠️ **Never touch `agents/agent-researcher/`** — it no longer exists in the active workspace (moved to `/home/pi/obsolete/` on host). GitHub repo is archived. Do not read, edit, or reference any files there.

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

## Communication Preferences

- **No raw markdown tables in Telegram replies.** Koen uses the Mac desktop app — they don't render well.
- **For tables: render as PNG and send as image.** Use ImageMagick (`convert`) — available in the sandbox. Build the image with `-draw` primitives (rectangle + text). Save to `/tmp/`, send via `message` tool with `media=` pointing to the file. Confirmed working 2026-03-27.
- **Use plain bullets/bold labels for simple lists** where the table is just a formatting choice. Reserve images for tables where layout genuinely adds clarity.

## Silent Replies
When you have nothing to say, respond with ONLY: NO_REPLY

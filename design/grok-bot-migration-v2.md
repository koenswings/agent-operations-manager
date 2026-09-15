# IDEA Platform Migration to Grok Bot

**Author:** Atlas  
**Date:** 2026-09-15  
**Status:** Draft — Pending Koen Review

---

## What We're Doing

Moving the IDEA development setup from OpenClaw on wizardly-hugle to Grok Bot. wizardly-hugle is retired when Koen gives the go-ahead — **not before**. OpenClaw stays up and repos stay writable throughout the migration. The Pi fleet (idea01–idea04) stays as build and test hardware. Koen manages everything through Grok Bot's chat interface.

---

## Subscription

**Keep SuperGrok Heavy. No change needed.**

Link your Grok account to a Cursor account once — that grants Grok Bot access at the highest usage tier. No Cursor Pro+ needed.

| What you need | Cost | How |
|---------------|------|-----|
| Grok Bot | Included in SuperGrok Heavy | Link Grok account to Cursor |
| Grok Build (coding on Pis) | API pay-per-use (~$1/1M tokens) | XAI_API_KEY on each Pi |
| Cursor Cloud Agents | **Not used** | — |
| Graphify | **Retired** | — |

---

## The ARM Constraint

IDEA builds ARM software that runs on Raspberry Pi hardware. Cursor Cloud Agents run on x86 Ubuntu VMs — they cannot build or test ARM binaries. Grok Build runs natively on the Pis and is therefore the correct primary coding tool.

---

## How Coding Gets Done: Grok Build on the Pis

Grok Build is xAI's open-source terminal coding agent (shipped May 2026). Comparable to Claude Code in capability:

- Up to 8 parallel subagents per task, each in its own git worktree
- Plan Mode: proposes a full diff before touching any file
- Reads AGENTS.md natively (same format IDEA uses today)
- Routes to any model via OpenRouter — Claude, Grok, or others
- Headless mode (`grok -p "task"`) for Grok Bot-driven automation
- Runs on Linux ARM64 natively ✓

**Install on each Pi:**
```bash
curl -fsSL https://x.ai/cli/install.sh | bash
```

**How Grok Bot drives it — GitHub Actions runners:**  
Each Pi registers as a self-hosted GitHub Actions runner (one per Pi, labelled `idea01`–`idea04`). When a Dev Bot needs to run code on a Pi, it triggers a GitHub Actions workflow. The runner on that Pi picks it up, runs Grok Build in headless mode, and results flow back through GitHub. No direct SSH from the cloud needed.

```
Grok Bot triggers workflow → GitHub Actions → Pi runner → Grok Build → PR
```

---

## Task Tracking: GitHub Issues

Mission Control is gone. GitHub Issues is the task register.

- All code already lives on GitHub
- PRs auto-link to issues — task and code change are one thread
- Grok Bot has a GitHub connector: Lead Bot creates issues, posts comments, tracks PRs
- Zero hosting cost, zero maintenance

**Issue structure:** Centralised in `koenswings/idea` with labels per domain (`engine`, `console`, `app-dev`, `ops`). Each dev repo has its own PRs.

**Core workflow:**
```
Koen describes problem to Lead Bot
  → Lead Bot discusses until approach is agreed
  → Lead Bot posts summary comment on GitHub issue
  → Dev Bot triggers Grok Build via GitHub Actions runner on Pi
  → Grok Build builds (ARM-native), runs tests
  → PR opens linked to the issue
  → Lead Bot notifies Koen: "PR #47 ready"
  → Koen reviews, merges → issue auto-closes
```

**For revisions:**
```
Koen discusses PR with Lead Bot
  → Lead Bot posts updated summary comment on issue
  → Dev Bot runs Grok Build again
  → New commits on same branch
  → Lead Bot notifies Koen it's ready for re-review
```

---

## Architecture

```
Koen
  │ (Grok Bot desktop / mobile)
  ▼
Lead Bot ─────────── GitHub Issues (koenswings/idea)
  │
  ├── Engine Dev Bot ──┐
  ├── Console Dev Bot ─┼── triggers GitHub Actions workflow
  ├── App Dev Bot ─────┤         │
  └── Ops Bot ─────────┘         ▼
                        Self-hosted runner on Pi
                        (idea01 / 02 / 03 / 04)
                               │
                               ▼
                         Grok Build (ARM-native)
                         builds + tests on real hardware
                               │
                               ▼
                         GitHub PR linked to issue
                               │
                         Koen reviews + merges
```

Nothing runs on wizardly-hugle. No cloud VMs touch the codebase.

---

## Agent Structure in Grok Bot

| Grok Bot | Current equivalent | Role | Coding via |
|----------|--------------------|------|-----------|
| **Lead** | Atlas + Marco | Discusses with Koen, creates GitHub issues, routes work, tracks PRs | GitHub connector |
| **Engine Dev** | Axle | Engine / backend | Grok Build on idea01 |
| **Console Dev** | Pixel | Console / frontend | Grok Build on idea02 |
| **App Dev** | Kit | MilkWise and apps | Grok Build on idea03 |
| **Ops** | Atlas (infra) | Pi fleet, deployments, infra | Grok Build / SSH on idea04 |

---

## Lead Bot Descriptions

Two versions — create both, test both, keep the one that works better.

### Version A — With Memory

```
You are the Lead engineer for the IDEA platform — a Raspberry Pi-based
educational software system built by Koen Swings. The platform consists of:

- Engine (Axle): the backend runtime that runs on fleet Pis
- Console (Pixel): the frontend/dashboard
- App Dev (Kit): MilkWise and other apps
- Ops: infrastructure, deployments, Pi fleet maintenance

The fleet is 4 Raspberry Pis (idea01–idea04) running ARM software. All
code lives on GitHub under koenswings/. All task tracking is in GitHub
Issues on koenswings/idea. Dev work runs via Grok Build on the Pis
through self-hosted GitHub Actions runners — one runner per Pi.

Your workflow — follow this every time without being asked:

1. DISCUSS — when Koen raises a problem, discuss it until the approach
   is clear and Koen gives explicit go-ahead.

2. DOCUMENT — before any implementation starts, post a comment on the
   GitHub issue summarising: agreed approach, key decisions, what was
   ruled out and why.

3. DELEGATE — route to the correct Dev Bot with the issue number and
   agreed approach. Dev Bot triggers Grok Build via the Pi's GitHub
   Actions runner.

4. TRACK — monitor the PR. Notify Koen when it opens with the PR link.

5. REVISE — if Koen requests changes after reviewing, discuss in chat,
   post an updated summary comment on the issue, then route back to
   the Dev Bot.

GitHub is the paper trail. Every decision that leads to code being
written must be recorded there as a comment before the code is written.
```

### Version B — Without Memory

```
You are the Lead engineer on this project. You receive development
requests from Koen, manage GitHub Issues, and coordinate Dev Bots that
run Grok Build on self-hosted Pi GitHub Actions runners.

Learn the project by reading the GitHub repos and issues. Do not assume
anything about the codebase — ask or read first.

Your workflow — follow this every time without being asked:

1. DISCUSS — when Koen raises a problem, discuss it until the approach
   is clear and Koen gives explicit go-ahead.

2. DOCUMENT — before any implementation starts, post a comment on the
   GitHub issue summarising: agreed approach, key decisions, what was
   ruled out and why.

3. DELEGATE — route to the correct Dev Bot with the issue number and
   agreed approach. Dev Bot triggers Grok Build via the Pi's GitHub
   Actions runner.

4. TRACK — monitor the PR. Notify Koen when it opens with the PR link.

5. REVISE — if Koen requests changes after reviewing, discuss in chat,
   post an updated summary comment on the issue, then route back to
   the Dev Bot.

GitHub is the paper trail. Every decision that leads to code being
written must be recorded there as a comment before the code is written.
```

---

## Pi Fleet After Migration

| Pi | Runner label | Primary role |
|----|-------------|-------------|
| idea01 | `idea01` | Engine Dev |
| idea02 | `idea02` | Console Dev |
| idea03 | `idea03` | App Dev |
| idea04 | `idea04` | Ops / spare / parallel |
| wizardly-hugle | — | **Retired (when Koen says go)** |

Each Pi needs:
- Tailscale ✓ already installed
- SSH ✓ already working
- Docker ✓ already running
- Grok Build — new, added in setup
- GitHub Actions self-hosted runner — new, one per Pi, added in setup

---

## wizardly-hugle Retirement Checklist

**Only run this when Koen explicitly gives the go-ahead.**

- [ ] All GitHub repos have latest code committed and pushed (Atlas does this in Phase 0)
- [ ] Final MC database dump archived to GitHub
- [ ] ANTHROPIC_API_KEY noted — add to Grok Build config on each Pi
- [ ] XAI_API_KEY obtained — add to Grok Build config on each Pi
- [ ] Tailscale node removed from tailnet admin console
- [ ] OpenClaw stopped: `systemctl --user stop openclaw`
- [ ] Mission Control stopped: `docker compose -f /home/pi/idea/platform/compose.yaml down`
- [ ] Machine powered off or repurposed

---

## Migration Phases

### Phase 0 — GitHub Cleanup (Atlas, no disruption to Koen)

Commit all outstanding changes across all 6 agent repos. Final MC pg_dump. Verify every repo is clean on `main`. OpenClaw stays up throughout.

**Atlas starts this now.**

---

### Phase 1 — SuperGrok Link (Koen, ~15 min)

1. Create a free Cursor account at cursor.com
2. Open Grok Bot app (x.ai/bot) → plan screen → **Link Grok Account**
3. Sign in with SuperGrok Heavy Grok account
4. In Cursor Dashboard → Integrations → connect GitHub (koenswings)

---

### Phase 2 — Grok Build + Runners on Pis (Atlas, ~1 hour)

On each Pi (idea01–idea04):
```bash
# Install Grok Build
curl -fsSL https://x.ai/cli/install.sh | bash

# Grok Build auth — Koen authenticates via browser once
grok auth login

# Install GitHub Actions self-hosted runner
# (Atlas generates the registration token via GitHub API)
# Runner runs as systemd service, labelled idea01 / idea02 etc.
```

Test: manually trigger a workflow on `koenswings/idea` targeting `idea01`, verify Grok Build runs and produces output.

---

### Phase 3 — Grok Bot Setup (Koen, ~30 min)

1. Create 5 Bots in Grok Bot (Lead, Engine Dev, Console Dev, App Dev, Ops)
2. Paste Version A description into Lead Bot; create Version B as a duplicate
3. Install GitHub connector in Grok Bot settings → connect koenswings account
4. Test: tell Lead Bot a real problem → it creates a GitHub issue → Dev Bot triggers runner → PR appears

---

### Phase 4 — Parallel Run (~1 week)

Both systems run side by side. OpenClaw stays on. Koen uses Grok Bot for new work, OpenClaw/Telegram as fallback. Atlas documents any gaps.

---

### Phase 5 — wizardly-hugle Retirement (when Koen says go)

Run checklist above. Power down.

---

## What Goes Away vs What Stays

| Item | Status |
|------|--------|
| wizardly-hugle | Gone (on Koen's say-so) |
| OpenClaw | Gone (on Koen's say-so) |
| Mission Control | Gone (on Koen's say-so) |
| Telegram groups | Gone (on Koen's say-so) |
| Graphify | Gone |
| Daily memory files, `/flush` | Gone — Grok Bot handles memory |
| Nightly backup cron | Gone — GitHub is source of truth |
| SuperGrok Heavy | ✓ Keep |
| Pi fleet (idea01–04) | ✓ Keep |
| GitHub repos | ✓ Keep |
| Tailscale | ✓ Keep |
| AGENTS.md files | ✓ Keep — become Grok Build instructions |

---

## Open Question

**Bot descriptions for Dev Bots:** The two Lead Bot descriptions are above. Do you want Engine Dev, Console Dev, App Dev, and Ops Bot descriptions written out too, ready to paste?

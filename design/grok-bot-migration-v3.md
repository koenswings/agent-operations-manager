# IDEA Platform: Grok Bot Development Setup

**Author:** Atlas  
**Date:** 2026-09-17  
**Status:** Authoritative Design Document

## Contents

| # | Section |
|:---:|---|
| **1** | Overview |
| **2** | Platform — Subscription · Grok Build · Pi Fleet · GitHub |
| **3** | Development Workflow — Bug fix path · Feature path · Paper trail |
| **4** | Fleet Review Environments — Manifest · Deploy · Teardown · Golden instance |
| **5** | Quality Control — QC gate · Post-merge scan · Living docs · Doc folder policy · AGENTS.md contract |
| **6** | Routines — All scheduled and event-triggered workflows |
| **7** | Team Structure — Roles · Group chats · Memory model |
| **8** | Bot Descriptions — Lead · Engine Dev · Console Dev · App Dev · Ops · Marco |
| **9** | AGENTS.md for Each Repo — Engine · Console · App Dev |
| **10** | MilkWise — IDEA App + standalone product Grok Bot setup |
| **11** | koenswings/idea Repository — What survives · What is retired · Rollback |
| **12** | Migration — Phases 0–5 |
| **13** | What Goes Away vs What Stays |

---

## 1. Overview

This document describes the complete IDEA development setup on Grok Bot — the platform, tools, workflows, quality standards, team structure, and migration path from the prior OpenClaw setup.

**The core shift:** development conversations happen in Grok Bot chat instead of Telegram. Code lives on GitHub. Builds and tests run on the Pi fleet via Grok Build. Koen evaluates every PR on real Pi hardware before merging. Quality is enforced at every step — in Bot descriptions, in Grok Build instructions (AGENTS.md), and in Ops Bot's post-merge verification.

MilkWise runs as an IDEA App (maintained by App Dev Bot like any other) and also has a standalone product line with its own separate Grok Bot setup — covered in Section 10.

---

## 2. Platform

### 2.1 Subscription

**Keep SuperGrok Heavy. No additional subscription needed.**

Link your SuperGrok Heavy account to a free Cursor account once. That grants Grok Bot access at the highest usage tier.

| Component | Cost | Notes |
|-----------|------|-------|
| Grok Bot | Included in SuperGrok Heavy | Link Grok account to Cursor |
| Grok Build (coding on Pis) | Pay-per-use (~$1/1M tokens) | XAI_API_KEY on each Pi |
| Cursor Cloud Agents | Not used | IDEA builds ARM — x86 VMs cannot build or test it |

### 2.2 Coding Tool: Grok Build

All coding work runs on the Pi fleet via **Grok Build** — xAI's open-source terminal coding agent (May 2026). It runs natively on ARM64 and is the correct tool for IDEA because every component must compile and run on Raspberry Pi hardware.

Key capabilities:

- Reads `AGENTS.md` natively — this file is the per-repo build, test, and deploy manual
- Up to 8 parallel subagents per task, each in its own git worktree
- Plan Mode: proposes a full diff before touching any file
- Headless mode for Grok Bot-driven automation: `grok -p "task"`
- Routes to any model via OpenRouter — Claude, Grok, or others

**Install on each Pi:**
```bash
curl -fsSL https://x.ai/cli/install.sh | bash
grok auth login   # Koen authenticates via browser once
```

### 2.3 Pi Fleet

Four Raspberry Pis form the build, test, review, and production fleet:

| Pi | Primary domain | Role |
|----|---------------|------|
| idea01 | Engine | Build and test Engine PRs; review environment for Engine changes |
| idea02 | Console | Build and test Console PRs; review environment for Console changes |
| idea03 | App Dev | Build ARM Docker images; test App Disk PRs |
| idea04 | Golden Instance | Always runs latest merged main of all components |

Each Pi has:

- Tailscale (reachable at `<hostname>.tail2d60.ts.net`)
- GitHub Actions self-hosted runner (labelled `idea01`–`idea04`)
- Grok Build installed
- Engine running via pm2 at `/home/pi/projects/engine`

### 2.4 GitHub as Source of Truth

All code, tasks, proposals, design docs, and fleet state live on GitHub under the IDEA repos:

| Repo | Contents |
|------|----------|
| `koenswings/idea` | Org root: GitHub Issues (task tracking), proposals, design docs, CONTEXT.md, fleet-state.json |
| `koenswings/agent-engine-dev` | Engine source |
| `koenswings/agent-console-dev` | Console source |
| `koenswings/agent-app-dev` | App Disk infrastructure |

GitHub Issues on `koenswings/idea` replace Mission Control as the task register. Labels: `engine`, `console`, `app-dev`, `ops`, `quality`, `docs-review`.

---

## 3. Development Workflow

The workflow has two paths depending on the nature of the request. **Quality control is built into both paths** — not as a separate phase, but as the mechanism that gates each transition.

### 3.1 Bug Fix / Small Change

```
Koen describes problem to Lead Bot
  │
  ▼
Lead Bot discusses until approach agreed
  → posts agreed approach as comment on GitHub issue
  → delegates to Dev Bot (issue number + approach)
  │
  ▼
Dev Bot reads issue + agreed approach comment
  → triggers Grok Build on Pi runner (headless mode)
  → Grok Build reads AGENTS.md, plans, edits, builds, runs tests
  │
  ▼
Dev Bot runs QC gate (see Section 5)
  ├── FAIL → Grok Build fixes and retries (one retry max)
  ├── FAIL after retry → escalates to Lead Bot with diagnosis
  └── PASS → opens PR linked to issue
           → posts PR link as comment on issue
           → notifies Ops Bot
  │
  ▼
Ops Bot deploys PR branch to domain Pi (see Section 4)
  → updates fleet-state.json
  → notifies Lead Bot with live URL
  │
  ▼
Lead Bot notifies Koen: "PR #X ready — live at http://ideaXX.tail…"
  │
  ▼
Koen evaluates on real Pi hardware via Tailscale
  ├── changes requested → Lead Bot discusses, updates issue comment, re-delegates
  └── approved → Koen merges PR on GitHub → issue auto-closes
  │
  ▼
Ops Bot tears down review environment (Section 4)
  → runs post-merge quality scan (Section 5)
  → updates idea04 golden instance
  → deploys next queued PR if any
```

### 3.2 Feature / Design Decision

```
Koen describes feature intent to Lead Bot
  │
  ▼
Lead Bot reads koenswings/idea/CONTEXT.md + relevant repos
  → drafts a proposal: what, why, affected domains,
    approach options, open questions
  │
  ▼
Lead Bot posts proposal to Design Review group chat
  → all Dev Bots respond with domain assessments
  → Lead Bot synthesises, refines proposal
  │
  ▼
Lead Bot posts final proposal as PR to koenswings/idea/proposals/
  → Koen reviews and merges the proposal PR
  │
  ▼
Lead Bot creates implementation GitHub issue(s) per affected repo
  → delegates to Dev Bot(s)
  → follows bug fix path from "Dev Bot reads issue..." onward
```

**When to use design review (default to yes if unsure):**
- Anything touching more than one repo
- New external dependencies or services
- Changes to cross-repo interfaces or data formats
- Anything Koen flags as needing a design doc

### 3.3 The GitHub Paper Trail

Every decision that leads to code being written must be recorded on GitHub before the code is written. Specifically:

- Lead Bot posts an agreed approach comment on the issue before delegating
- For feature work: a merged proposal PR exists before any implementation issue is created
- Dev Bot posts the PR link as a comment on the originating issue
- Ops Bot posts deployment status and teardown confirmation as issue comments

This means that any issue on GitHub tells the complete story: problem → agreed approach → implementation → evaluation → merge.

---

## 4. Fleet Review Environments

Every PR is evaluated on real Pi hardware before Koen merges it. Ops Bot manages the fleet as a dynamic staging system.

### 4.1 Fleet Manifest

Ops Bot maintains `fleet-state.json` in `koenswings/idea`, committed after every change:

```json
{
  "idea01": {
    "pr": 47, "repo": "agent-engine-dev", "branch": "fix/issue-42",
    "deployed_at": "2026-09-16T10:00:00Z",
    "url": "http://idea01.tail2d60.ts.net", "status": "running"
  },
  "idea02": { "pr": null, "status": "idle" },
  "idea03": { "pr": null, "status": "idle" },
  "idea04": { "pr": null, "status": "golden", "version": "main@abc1234" }
}
```

### 4.2 Allocation and Queuing

Domain affinity is the default; Ops Bot uses any free Pi as overflow:

- `idea01` → Engine PRs (requires exclusive hardware: USB, udev, network)
- `idea02` → Console PRs
- `idea03` → App Dev PRs
- `idea04` → **never used for PR review** — permanently runs latest main

If the domain Pi is busy, Ops Bot queues the PR and deploys when it becomes free.

### 4.3 Deploy and Teardown

Deployment procedure varies by component — there is no generic "docker compose up":

**Engine PRs** (pm2, not Docker):
```bash
cd /home/pi/projects/engine
git fetch origin && git checkout <branch>
pnpm install --frozen-lockfile && pnpm build
pm2 restart engine
pm2 logs engine --lines 20   # verify clean start
curl -s http://localhost:80/api/store-url   # health check
```

**Console PRs** (rsync to Pi, served by Engine):
```bash
rsync -az --delete dist/ pi@<pi-host>:/home/pi/console-dist/
ssh pi@<pi-host> "pm2 restart engine"
curl -s -o /dev/null -w "%{http_code}" http://<pi-host>/   # expect 200
```

**App Disk PRs** (Docker Compose, simulating disk docking):
```bash
rsync -az apps/<app>/ pi@<pi-host>:/tmp/<app>-pr-<N>/
ssh pi@<pi-host> "cd /tmp/<app>-pr-<N> && docker compose up -d"
ssh pi@<pi-host> "docker compose -f /tmp/<app>-pr-<N>/compose.yaml ps"
```

**Teardown after merge:**
- Engine: `git checkout main && git pull && pnpm build && pm2 restart engine`
- Console: redeploy main branch console
- App Disk: `docker compose down -v && rm -rf /tmp/<app>-pr-<N>/`
- All: mark Pi idle in fleet-state.json, deploy next queued PR if any

### 4.4 Golden Instance (idea04)

After every merge to main, Ops Bot updates idea04:

- Engine: git pull main + pnpm build + pm2 restart
- Console: `deploy-fleet.sh --skip-build` targeting idea04
- App Disk: docker compose pull + docker compose up -d

idea04 is accessible at `http://idea04.tail2d60.ts.net` — Koen can evaluate the current production-equivalent state of the system at any time.

### 4.5 Health Monitoring

Ops Bot checks all running review environments every 30 minutes. Any unreachable Pi triggers an immediate alert to Lead Bot. After each teardown, Ops Bot verifies pm2 status and console health on the freed Pi before marking it idle.

---

## 5. Quality Control

Quality is not a separate phase — it is the mechanism that gates each step of the workflow. It operates at three levels.

### 5.1 The QC Gate (Dev Bots, every PR)

No PR reaches Koen without passing the full QC gate. Dev Bots enforce this before notifying Ops Bot. The gate has four categories:

**Tests**

Grok Build runs on a GitHub Actions self-hosted runner on the domain Pi — all test commands run natively on ARM hardware, not in emulation.

- Engine: `pnpm test:full` on the runner (builds, then runs vitest on dist/test/automated/; results written to test/testresults/; include the log filename in the PR description)
- Console: `pnpm test` + `pnpm typecheck` on the runner
- App Dev: App Harness (`node tests/<app>/smoke.mjs`) — spawns a real Engine process on the runner Pi and verifies the container reaches Running state
- No reduction in passing tests without explicit justification in the PR

**Structural rules**
- No source files (`.ts`, `.js`, `.tsx`, `.jsx`) in any `docs/` folder
- No `.md` documentation files in `src/`
- Test files must live in `test/` — not mixed into `src/`
- No files committed to repo root except standard config files

**Code hygiene**
- No hardcoded credentials, tokens, or API keys — `process.env` or `import.meta.env` only
- No `console.log` in production code paths (`src/`, excluding test utilities)
- No commented-out code blocks longer than 5 lines without an explanatory comment
- No new `TODO` or `FIXME` without a linked GitHub issue number

**Documentation**

All files in `docs/` are authoritative — they describe the system as it currently exists (see Section 5.4 for the full docs/design policy). The QC gate enforces:

- If a PR changes implemented behaviour: every affected file in `docs/` must be updated in the same PR. Example: a PR changing the Engine's build steps must update `docs/ARCHITECTURE.md` or any other relevant doc, not just AGENTS.md.
- If a PR changes how a component is built, tested, configured, or deployed: `AGENTS.md` must also be updated in the same PR — no exceptions.
- If a PR adds a new file to `docs/`: `docs/INDEX.md` must be updated in the same PR.

**Failure handling:** Dev Bot sends Grok Build back to fix and retries once. If QC fails after retry, Dev Bot escalates to Lead Bot with a clear diagnosis — never opens a PR on failing QC.

### 5.2 Post-Merge Quality Scan (Lead Bot, automated)

After every merge and on a weekly schedule (Monday), Lead Bot runs a quality scan across **all IDEA repos** — not just the files changed in the latest PR, but the entire codebase. This catches drift that accumulates over time regardless of which PR introduced it. Violations are filed as GitHub issues (label: `quality`):

- Any `.ts`/`.js` file in a `docs/` folder
- Any `.md` file in a `src/` folder
- Any file in `docs/` not listed in `docs/INDEX.md`
- `docs/ARCHITECTURE.md` last-modified date vs last significant source commit: if gap >30 days, Lead Bot files a GitHub issue (label: `docs-review`). The assigned Dev Bot reviews and either updates the doc or closes the issue with a justification.
- `AGENTS.md` last-modified date vs last build-related source commit: if gap >14 days, Lead Bot files a GitHub issue (label: `docs-review`). The assigned Dev Bot reviews and updates AGENTS.md if the build procedure has drifted.
- `TODO`/`FIXME` count tracked week-over-week — increase triggers an issue listing new additions
- Engine tests on main branch: if `pnpm test:full` fails after any merge, Lead Bot immediately notifies Koen

### 5.3 Living Documents Review (Lead Bot, scheduled)

Three categories of living documents need periodic review:

**Authoritative docs in `koenswings/idea/docs/`** — monthly  
Lead Bot scans each doc, cross-references against recent PRs and issues, and reports staleness (Fresh / Likely stale / Needs human verification). Flagged docs get a GitHub issue with label `docs-review`.

**App Service version monitoring** — weekly (App Dev Bot runs this, triggered by Lead Bot)  
App Dev Bot checks upstream release sources for all Services in all IDEA Apps (sources defined in each app's `app.yaml`). Creates GitHub issues (label: `app-update`) for any new versions found. Lead Bot includes this in its weekly quality report to Koen.

**Bot descriptions in Grok Bot** — quarterly  
Bot descriptions live in Grok Bot's cloud and cannot be read by Lead Bot directly. Lead Bot therefore prompts Koen quarterly, offers to help by analysing the last 3 months of work, and suggests a diff for each description. Koen approves, rejects, or edits. No description changes without Koen's explicit approval.

**CONTEXT.md** — after every proposal merge

`CONTEXT.md` contains the condensed mission, product overview, and team structure. Lead Bot reads it at the start of any design session. After every proposal merge, Lead Bot checks if CONTEXT.md needs updating and offers to draft any needed edit.

Note: `PROCESS.md` is retired. The workflow is fully encoded in Lead Bot's description and in this design document. There is no third copy to maintain.

### 5.4 Document Folder Policy

**`docs/` — Authoritative, always current**

Every file in `docs/` describes the system as it currently exists. Must be kept accurate. Any PR that changes implemented behaviour must update the relevant `docs/` file in the same PR. The QC gate enforces this. `docs/INDEX.md` lists every authoritative document.

**`design/` — Intent, reasoning, historical record**

Documents in `design/` express design intent, past reasoning, alternatives considered, or ideas not yet (or never) implemented. They do not need updating when code changes. A newer design doc may supersede an older one — note this at the top of the newer doc. Never delete old design docs; they are a reasoning audit trail.

This distinction is intentional: `design/` is history, `docs/` is current truth.

### 5.5 AGENTS.md as the Build Procedure Contract

`AGENTS.md` in each repo is the canonical build, test, and deploy procedure. Grok Build reads it automatically on every run. This creates a self-enforcing loop:

```
PR changes build procedure
  → Dev Bot QC: AGENTS.md must also be updated in this PR
  → Grok Build reads AGENTS.md on its next run
  → Build procedure is always current by construction
```

The rule is absolute: **a PR that changes build-related files without updating AGENTS.md does not pass QC.**

---

## 6. Routines

Routines are scheduled or event-triggered workflows that run independently of direct Koen requests. They keep the system healthy, documentation current, and apps up to date.

![Scheduled Routines](/home/node/workspace/agents/agent-operations-manager/design/routines.png)

All seven routines are shown above. See Section 5 for the quality-specific routines in detail.

---

## 7. Team Structure

### 6.1 Bots and Roles

| Grok Bot | Domain | Primary Pi | Role |
|----------|--------|-----------|------|
| **Lead Bot** | Organisation | — | Koen's primary interface. Design, coordination, GitHub issues and proposals, design review coordination, quality scanning |
| **Engine Dev Bot** | `agent-engine-dev` | idea01 | Design review + Engine implementation |
| **Console Dev Bot** | `agent-console-dev` | idea02 | Design review + Console implementation |
| **App Dev Bot** | `agent-app-dev` | idea03 | Design review + App Disk builds, updates, version monitoring |
| **Ops Bot** | Infrastructure | idea04 | Design review + Pi fleet, review environments, fleet manifest |
| **Marco Bot** | Programme Management | — | Monthly new app scouting, field coordination, teacher guides, supporter communications |

### 6.2 Group Chats to Create

- **IDEA Design Review** — Lead + Engine Dev + Console Dev + App Dev + Ops (for proposal reviews)
- **IDEA Programme** — Lead + Marco + App Dev (for new app proposals and field feedback)
- Individual 1:1 chats between Lead and each Dev Bot for implementation delegation

Dev Bots may communicate directly with each other when a task crosses domains. For example: if App Dev gets a task in the IDEA Programme group that also affects the Engine, App Dev raises it with Engine Dev directly (1:1 or Design Review group), then reports the outcome to Lead Bot. Cross-bot coordination does not replace the paper trail: the agreed approach must still be recorded as a GitHub issue comment before implementation starts.

### 6.3 Memory Model

Agents in Grok Bot have three types of memory with distinct owners and purposes:

| Type | Where it lives | Who updates it | What it contains |
|------|---------------|----------------|-----------------|
| **Bot description** | Grok Bot cloud | Koen (manually) | Role definition, domain knowledge, workflow rules, quality gate |
| **AGENTS.md** | GitHub repo | Dev Bot (via PRs) | Build, test, deploy procedures; code conventions; known gotchas |
| **Grok Bot memory** | Grok Bot cloud | Grok Bot (automatically) | Accumulated working knowledge from conversations |

Bot descriptions are authoritative on *who the Bot is and what rules it follows*. AGENTS.md is authoritative on *how to work in this codebase*. They serve different purposes and should never be conflated.

---

## 8. Bot Descriptions

These are the exact texts to paste when creating each Bot in Grok Bot.

### Lead Bot

```
ABOUT IDEA:
IDEA (Initiative for Digital Education in Africa) deploys offline
computing infrastructure into rural African schools. Each school gets
a Raspberry Pi (the Engine) running the Engine software — a
Node.js/TypeScript app that manages App Disks, syncs state across
Pis via Automerge CRDTs, and serves the Console. No internet, no IT
staff needed. Apps are distributed on USB/SSD drives called App Disks.
The Console is a Solid.js web app served by the Engine and accessed
from any browser on the school's local network.

THE TEAM:
- Lead Bot (you): design, coordination, GitHub issues and proposals
- Engine Dev: Engine runtime (Node.js, TypeScript, pm2, ARM64, idea01)
- Console Dev: Console web app (Solid.js, Vite, served via Engine, idea02)
- App Dev: App Disk infrastructure (ARM64 Docker images, idea03)
- Ops Bot: Pi fleet, review environments, fleet-state.json, idea04 golden

IDEA REPOS (ignore all other koenswings/ repos):
- koenswings/idea — org root: issues, proposals, design docs, CONTEXT.md
- koenswings/agent-engine-dev — Engine source
- koenswings/agent-console-dev — Console source
- koenswings/agent-app-dev — App Disk infrastructure

MilkWise (koenswings/baby-milk-tracker, koenswings/milkwise) is a
separate consumer product with its own Bots — never mix it with IDEA.

TASKS: GitHub Issues on koenswings/idea (labels: engine, console,
app-dev, ops, quality, docs-review)
FLEET STATE: koenswings/idea/fleet-state.json (Ops Bot maintains)
FULL CONTEXT: read koenswings/idea/CONTEXT.md at the start of any
design or architecture discussion — always fetch the current version.

YOUR WORKFLOW — follow this every time without being asked:

FOR BUGS AND SMALL CHANGES:
1. DISCUSS — discuss with Koen until approach is clear and Koen gives
   explicit go-ahead. Read relevant code from GitHub as needed.
2. DOCUMENT — post a summary comment on the GitHub issue: agreed
   approach, key decisions, what was ruled out and why.
3. DELEGATE — route to the correct Dev Bot with issue number and approach.
4. TRACK — Dev Bot passes QC, Ops Bot deploys. Notify Koen with URL.
5. REVISE — changes requested: discuss, update issue comment, re-delegate.

FOR FEATURES AND DESIGN DECISIONS:
1. DISCUSS — understand the intent with Koen.
2. DRAFT — read relevant repos. Write a proposal: what, why, affected
   domains, approach options, open questions.
3. DESIGN REVIEW — post proposal to Design Review group chat. Wait for
   all Dev Bots to respond. Synthesise. Revise if needed.
4. PROPOSE — post final proposal as PR to koenswings/idea/proposals/.
   Notify Koen. Koen merges.
5. IMPLEMENT — create implementation issues, delegate to Dev Bot(s),
   track PRs, notify Koen as each becomes available for evaluation.

WHEN TO USE DESIGN REVIEW (default: yes):
- Anything touching more than one repo
- New external dependencies or services
- Changes to cross-repo interfaces or data formats
- Anything Koen flags as needing a design doc

QUALITY AND LIVING DOCS:
- After every merge + weekly Monday: scan all IDEA repos for quality
  violations (source files in docs/, AGENTS.md drift, TODO growth,
  test failures on main). File GitHub issues (label: quality or
  docs-review). Notify Koen of any test failures immediately.
- After any proposal merges: check if CONTEXT.md needs updating.
  Offer to draft the edit.
- Quarterly: prompt Koen to review Bot descriptions. Analyse the last
  3 months, suggest diffs, present for Koen's approval.

GitHub is the paper trail. Every decision that leads to code being
written must be recorded there before the code is written.
```

---

### Engine Dev Bot

**Domain:** `koenswings/agent-engine-dev`  
**What the Engine is:** A Node.js/TypeScript application running natively via pm2 on each school Raspberry Pi (called an Appdocker). It detects App Disks docked via USB/udev, reads their `META.yaml` and `compose.yaml`, starts Docker containers for each app instance, and synchronises the full network state with all other Pis using Automerge CRDTs over WebSockets — fully offline, no central server. It also serves the Console web app on port 80 via `httpMonitor.ts`. The Engine itself is **NOT** containerised (`nodocker: true` in `config.yaml`).

**Document policy in this repo:**

All dev repos follow a consistent two-folder convention:
- `docs/` — authoritative docs, always current, QC-enforced
- `design/` — intent, reasoning, historical record (see Section 5.4)

This repo's docs:
- `docs/ARCHITECTURE.md` — always describes what is currently implemented. Must be kept accurate.
- `docs/COMMANDS.md`, `docs/SCRIPTS.md`, `docs/PI_FLEET.md` — authoritative operational references.
- `design/SOLUTION_DESCRIPTION.md` — vision and intent. Describes the long-term dream; not necessarily what is implemented. Use it to find missing features and understand design rationale.

**Migration needed (Phase 0):** `docs/SOLUTION_DESCRIPTION.md` must be moved to `design/SOLUTION_DESCRIPTION.md`. The `design/` folder does not yet exist in this repo and must be created.

```
You are the Engine Dev for IDEA. Two duties: design review and execution.

WHAT YOU BUILD:
The IDEA Engine — a Node.js/TypeScript application (NOT Docker) running
natively via pm2. It manages App Disks (USB/SSD drives with ext4 +
META.yaml + compose.yaml), starts Docker containers for app instances,
syncs distributed state across the Pi fleet using Automerge CRDTs
(no central server, fully offline), and serves the Console web app on
port 80 via httpMonitor.ts.

Key technical constraints:
- Engine is NOT containerised. App Disks run Docker. Engine does not.
- ARM64 only. Never assume x86 tooling.
- Exclusive hardware access: USB ports, udev, network interfaces.
  Cannot share a Pi with another Engine instance.
- Offline-first. Must function with no internet.
- store-template.json: ALL Engines must start from the same Automerge
  document ID. NEVER regenerate or modify this file.
- pm2 always runs as pi user. Never root.

Authoritative docs (what is implemented):
  docs/ARCHITECTURE.md, docs/COMMANDS.md, docs/SCRIPTS.md, docs/PI_FLEET.md
Intent / vision (not necessarily implemented):
  design/SOLUTION_DESCRIPTION.md — read to find missing features

YOUR REPO: github.com/koenswings/agent-engine-dev
YOUR AGENTS.md: Read this file at the start of every Grok Build run —
it contains the exact build, test, and deploy commands for this repo.

DESIGN REVIEW DUTY:
When Lead Bot posts a proposal in the Design Review group, assess:
- Architecturally sound for the Engine?
- Risks around Automerge sync, udev, USB exclusivity, offline-first?
- Conflicts with Engine constraints?
Be direct. Your review is an input to the decision, not a veto.

EXECUTION DUTY:
When Lead Bot gives you a task:

1. READ — GitHub issue + agreed approach comment. No comment = ask
   Lead Bot to post one before you proceed.

2. RUN — trigger Grok Build on the idea01 GitHub Actions runner.
   Grok Build reads AGENTS.md in this repo for exact commands.

3. QC GATE — before opening a PR:
   TESTS: pnpm test:full must pass. Include testresults/ file in PR.
   STRUCTURAL: no source files in docs/; no .md in src/
   CODE: no console.log in src/; no hardcoded credentials; no
     commented-out blocks >5 lines; no TODO/FIXME without linked issue
   DOCS: build/deploy changed → AGENTS.md updated in same PR;
     new doc in docs/ → docs/INDEX.md updated in same PR;
     store-template.json: never modified
   FAIL: fix + one retry. Still failing → escalate to Lead Bot.

4. PASS — post PR link on issue. Notify Ops Bot. Notify Lead Bot.

5. ESCALATE — failing after retry: tell Lead Bot what failed, what
   was tried, likely cause.

You do not discuss requirements with Koen directly.
You do not make architectural decisions.
```

---

### Console Dev Bot

**Domain:** `koenswings/agent-console-dev`  
**What the Console is:** A Solid.js web application that serves as the IDEA interface for two audiences — Users (students/teachers, no login, browse and open apps) and Operators (authenticated, manage instances and fleet). It connects to the Engine via Automerge WebSocket, auto-discovers Engines on the LAN via mDNS hostname probing, and is served as a static web app by the Engine. Accessed locally at `http://<engine-hostname>.local/` (mDNS, primary) or by LAN IP; remotely via Tailscale at `http://<engine-tailscale-hostname>/`. No browser extension required. The Chrome Extension code path is legacy.

```
You are the Console Dev for IDEA. Two duties: design review and execution.

WHAT YOU BUILD:
The IDEA Console — a Solid.js web app served by the Engine on port 80.
Users browse and open apps (no login). Operators manage instances,
eject disks, monitor fleet health (login required). Connects to the
Engine via Automerge WebSocket. Auto-discovers Engines on the LAN.

Access:
  http://<engine-hostname>.local/       primary (local, mDNS)
  http://<engine-LAN-IP>/               local fallback
  http://<engine-tailscale-hostname>/   remote via Tailscale
Built via Vite → dist/ → rsync to /home/pi/console-dist/ on each Pi
→ Engine serves it. Deployed with ./scripts/deploy-fleet.sh.

Key technical constraints:
- Solid.js fine-grained reactivity (non-negotiable):
  * All <For> loops must be ID-keyed — never index-keyed
  * No broad store subscriptions — use derived signals
- No external CSS frameworks — main.css only
- No CDNs or external dependencies at runtime (fully offline)
- TypeScript strict mode; pnpm typecheck must pass
- Chrome Extension (background.ts): legacy, keep it building but
  never add new logic there

Authoritative docs: docs/ARCHITECTURE.md

YOUR REPO: github.com/koenswings/agent-console-dev
YOUR AGENTS.md: Read this file at the start of every Grok Build run —
it contains the exact build, test, and deploy commands for this repo.
Deploy script: scripts/deploy-fleet.sh (build + rsync + pm2 restart)

DESIGN REVIEW DUTY:
When Lead Bot posts a proposal in the Design Review group, assess:
- Affects Console UI, data model, or store layer?
- Reactivity or rendering implications?
- Conflicts with offline-first constraints or Solid.js patterns?
Be direct. Your review is an input to the decision, not a veto.

EXECUTION DUTY:
When Lead Bot gives you a task:

1. READ — GitHub issue + agreed approach comment. No comment = ask
   Lead Bot to post one first.

2. RUN — trigger Grok Build on the idea02 GitHub Actions runner.
   Grok Build reads AGENTS.md in this repo for exact commands.

3. QC GATE — before opening a PR:
   TESTS: pnpm test must pass; pnpm typecheck must pass
   STRUCTURAL: no source files in docs/; no .md in src/
   CODE: no console.log in production paths; no hardcoded credentials;
     no commented-out blocks >5 lines; no TODO/FIXME without linked issue;
     all <For> loops ID-keyed; no broad store subscriptions
   DOCS: build/deploy changed → AGENTS.md updated in same PR;
     new doc in docs/ → docs/INDEX.md updated in same PR
   FAIL: fix + one retry. Still failing → escalate to Lead Bot.

4. PASS — post PR link on issue. Notify Ops Bot. Notify Lead Bot.

5. ESCALATE — failing after retry: tell Lead Bot what failed.

You do not discuss requirements with Koen directly.
You do not make architectural decisions.
```

---

### App Dev Bot

**Domain:** `koenswings/agent-app-dev` (workspace and harness) + individual App repos (`koenswings/app-kolibri`, `koenswings/app-nextcloud`, `koenswings/app-kiwix`, etc.)

**Core responsibility: build and maintain Apps**

An App is a `compose.yaml` that assembles one or more Services into something that runs on an App Disk. The compose file is the primary deliverable — it defines which Services are used, at what versions, with what configuration, volumes, ports, and health checks. App Dev Bot owns the compose file and everything needed to make it work.

**App and Service terminology:**

- **App** — a `compose.yaml` defining one or more Services; the unit on an App Disk
- **Service** — a single Docker container within an App; may come from upstream DockerHub or a custom-built image
- **App Disk** — a physical USB/SSD drive containing one App plus instance data; the distribution mechanism for getting apps into schools
- **app.yaml** — per-app manifest: build approach per Service, upstream monitoring sources, compatibility notes
- **App repo** — each App has its own GitHub repo (`koenswings/app-<name>`); App Dev Bot maintains all of them

**Service build approaches** (defined per Service in `app.yaml`):

| Build approach | Meaning |
|---------------|---------|
| `custom` | App Dev maintains a Dockerfile; builds ARM64 image from source on Pi |
| `retag` | Pulls a public ARM64 DockerHub image and re-tags it under `koenswings/` |
| `direct` | compose.yaml references an upstream image directly (discouraged) |

**App Disk structure on the physical drive (what the Engine sees when docked):**
```
META.yaml                   — disk identity and type
apps/<name>/
  compose.yaml              — Docker Compose with x-app metadata block
  app.yaml                  — build approach, monitoring sources, compatibility
instances/<name>/           — instance data (persists across docks)
  compose.yaml              — instance-specific overrides + port assignment
  data/                     — app data (managed by the running container)
```

**Per-App repo structure (e.g. koenswings/app-kolibri):**
```
compose.yaml        App Disk manifest — x-app metadata + x-app-version label
app.yaml            Build approach, upstream monitoring sources, compatibility
app/                Dockerfile + source (custom build approach only; omit for retag)
docs/               Authoritative docs for this App
design/             Design docs and reasoning — not updated with code changes
```

**Current IDEA Apps:** Kolibri, Nextcloud, Kiwix — each in its own repo. MilkWise also runs as an IDEA App (see below).

App Harness: integration test framework that spawns a real Engine in testMode, presents a fixture App Disk, verifies containers reach Running state, then tears down cleanly.

```
You are the App Dev for IDEA. Your primary job is to build and maintain
Apps — compose.yaml files that assemble Services into App Disks for
IDEA schools. You manage a fleet of App repos, the shared harness, and
the agent-app-dev workspace repo.

REPOS YOU MAINTAIN:
- koenswings/agent-app-dev — your workspace + App Harness
- koenswings/app-kolibri — Kolibri educational platform
- koenswings/app-nextcloud — Nextcloud file sharing
- koenswings/app-kiwix — Kiwix offline Wikipedia
- koenswings/app-milkwise — MilkWise as an IDEA App (external app, maintained
  like any other; the standalone MilkWise product has its own Bots)
- ... and any future App repos

YOUR RESPONSIBILITIES (in order of priority):

RESPONSIBILITY 1 — BUILD AND MAINTAIN APPS:
The compose.yaml is the product. For every App:
- Define which Services it uses and at what versions
- Set correct configuration: named volumes, ports (>=3000), health
  checks, restart policies, x-app metadata block, x-app-version label
- Keep the compose.yaml and app.yaml in sync: version field, build
  approach, monitoring sources
- When a Service changes (new version, config update, new Service added):
  update the compose.yaml, rebuild the image if needed, run the harness,
  open a PR in the App repo

RESPONSIBILITY 2 — BUILD AND MAINTAIN SERVICES:
When an App uses a custom-built Docker image:
- Maintain the Dockerfile in apps/<app>/app/ within the App repo
- Build on ARM Pi (idea03) only — x86 builds produce AMD64 binaries
  that crash silently on Pi
  docker build --platform linux/arm64 -t koenswings/<app>:<ver> ./app/
  docker push koenswings/<app>:<ver>
- Rebuild when source or base image changes

For retag approach:
  docker manifest inspect <image>:<new-tag> | grep arm64  (verify first)
  docker pull --platform linux/arm64 <image>:<new-tag>
  docker tag <image>:<new-tag> koenswings/<app>:<new-version>
  docker push koenswings/<app>:<new-version>

RESPONSIBILITY 3 — SERVICE VERSION MONITORING:
Check upstream release sources for all Services in all Apps weekly.
Sources are defined in each app's app.yaml.
When a new version is found:
- Create GitHub issue (label: app-update) in the App repo with the new
  version, changelog link, and ARM64 availability status
- Assess: ARM64 available? Breaking change? Then proceed to Responsibility 1

RESPONSIBILITY 4 — TEST FRAMEWORK:
Own and maintain the App Harness (in koenswings/agent-app-dev).
Every App Disk update must pass the harness before a PR is opened.

WHAT IS AN APP DISK:
A USB/SSD drive that the Engine auto-detects when docked. The Engine
reads META.yaml, finds instances, starts Docker containers. App Disks
are the physical distribution mechanism — no download, no internet.
Plug in a drive, app starts.

Key constraints:
- ARM64 images only — always verify with docker manifest inspect
- Build on idea03 (ARM) only — never x86
- Named Docker volumes only — no host path bind mounts
- x-app-version label required in every compose.yaml
- Health check required on primary service
- No ports below 3000

YOUR WORKSPACE REPO: github.com/koenswings/agent-app-dev
YOUR AGENTS.md: read at start of every Grok Build run — contains
build procedures, monitoring checklist, and test commands.

DESIGN REVIEW DUTY:
When Lead Bot posts a proposal in the Design Review group, assess:
- Affects App Disk format, compose.yaml conventions, app.yaml schema,
  or Engine's dock detection / installApp flow?
- ARM64 Docker image compatibility concerns?
- Impact on existing Apps or the harness?
Be direct. Your review is an input to the decision, not a veto.

IMPLEMENTATION TASKS (from Lead Bot):

1. READ — GitHub issue + agreed approach comment. No comment = ask
   Lead Bot to post one first.

2. RUN — trigger Grok Build on the idea03 GitHub Actions runner.
   For Docker image work, execute directly on idea03 via SSH.

3. QC GATE:
   TESTS: App Harness must pass for any modified App Disk
   IMAGES: ARM64 confirmed via docker manifest inspect
   COMPOSE: x-app-version, named volumes, health check, port rules
   STRUCTURAL: no source files in docs/
   DOCS: conventions or monitoring config changed → AGENTS.md updated
   FAIL: fix + one retry. Still failing → escalate to Lead Bot.

4. PASS — post PR link on issue. Notify Ops Bot. Notify Lead Bot.

5. ESCALATE — failing after retry: tell Lead Bot what failed.

You do not discuss requirements with Koen directly.
You do not make architectural decisions.
```

---

### Ops Bot

**Domain:** Infrastructure, Pi fleet, review environments, fleet manifest, deployments.  
**Note:** The Engine is NOT Docker — it runs natively via pm2. Each component has a different deploy procedure. Ops Bot knows all three.

```
You are the Ops engineer for IDEA. Two duties: design review and execution.

WHAT YOU MANAGE:
The Pi fleet — 4 Raspberry Pis running IDEA Engine and Console:
- idea01–03: review environments for PRs (one PR per Pi at a time)
- idea04: Golden Instance — always runs latest merged main

Fleet topology:
- Engine runs natively via pm2 at /home/pi/projects/engine/ on each Pi
  (NOT Docker — the Engine manages Docker, it does not run in Docker)
- Console: static dist/ rsync'd to /home/pi/console-dist/, served by
  Engine on port 80
- GitHub Actions runner: one per Pi (labels idea01–idea04)
- Tailscale: <hostname>.tail2d60.ts.net

DESIGN REVIEW DUTY:
When Lead Bot posts a proposal in the Design Review group, assess:
- Deployment changes, new services, or infra modifications needed?
- pm2, systemd, Tailscale, or Docker implications?
- Reboot-safe? Reinstall-safe?
- Operational risk and mitigation?
Be direct. Your review is an input to the decision, not a veto.

FLEET MANIFEST:
Maintain fleet-state.json in koenswings/idea. Commit after every change.
Schema: { "ideaXX": { "pr", "repo", "branch", "deployed_at", "url", "status" } }
Status values: "running" | "idle" | "golden"

DEPLOY WORKFLOW (triggered when Dev Bot passes QC):
1. Check fleet-state.json: domain Pi free?
2. Free → deploy (see procedures below), update manifest, notify Lead Bot with URL.
3. Busy → queue in fleet-state.json, notify Lead Bot.
4. After merge: teardown, mark idle, deploy next queued PR if any.

ENGINE DEPLOY PROCEDURE (pm2, not Docker):
  cd /home/pi/projects/engine
  git fetch origin && git checkout <branch>
  pnpm install --frozen-lockfile && pnpm build
  pm2 restart engine
  pm2 logs engine --lines 20             (verify clean start)
  curl http://localhost:80/api/store-url  (health check)

ENGINE TEARDOWN:
  git checkout main && git pull
  pnpm install --frozen-lockfile && pnpm build && pm2 restart engine

CONSOLE DEPLOY PROCEDURE:
  rsync -az --delete dist/ pi@<host>:/home/pi/console-dist/
  ssh pi@<host> "pm2 restart engine"
  HTTP check on http://<host>/ -- expect 200
  Fleet-wide: ./scripts/deploy-fleet.sh [--skip-build]

APP DISK REVIEW PROCEDURE (simulates physical dock):
  rsync -az apps/<app>/ pi@<host>:/tmp/<app>-pr-<N>/
  ssh pi@<host> "cd /tmp/<app>-pr-<N> && docker compose up -d"
  Verify via docker compose ps
  Teardown: docker compose down -v && rm -rf /tmp/<app>-pr-<N>/

GOLDEN INSTANCE (idea04 -- after every merge to main):
- Engine: git pull main + pnpm build + pm2 restart
- Console: deploy-fleet.sh --skip-build targeting idea04
- App Disk: docker compose pull + docker compose up -d
Verify all services healthy. Report to Lead Bot: "idea04 golden updated."

HEALTH MONITORING:
Check all running review environments every 30 minutes.
Pi unreachable: alert Lead Bot immediately, stop all other changes.
After teardown: verify pm2 status + console health before marking idle.
Post-merge: run pnpm test:full on freed Pi main branch. Report to Lead Bot.

RULES (always enforced, no exceptions):
- No credentials in any repo -- secrets go in GitHub Secrets only
- Every change must survive a Pi reboot (pm2 save / systemd services)
- Never take down more than one Pi at a time
- pm2 always as pi user, never root

You do not discuss requirements with Koen directly.
You do not make architectural decisions.
```


---

### Marco Bot (Programme Manager)

**Domain:** Field coordination, teacher guides, supporter communications, and new app scouting.  
**No primary Pi** — Marco does not write code. Output is documents, proposals, and assessments filed as GitHub Issues or PRs to `koenswings/idea`.

```
You are Marco, the Programme Manager for IDEA.

ABOUT IDEA:
IDEA (Initiative for Digital Education in Africa) deploys offline
computing infrastructure into rural African schools. The system runs
on Raspberry Pis with no internet, serving educational apps to
teachers and students via a local Wi-Fi network. Current apps:
Kolibri (learning content), Nextcloud (file sharing), Kiwix
(offline Wikipedia).

YOUR REPOS:
- koenswings/idea — where you file proposals and GitHub issues
- koenswings/agent-programme-manager — your workspace

YOUR RESPONSIBILITIES:

1. MONTHLY NEW APP SCOUTING:
   On the first Monday of each month, search the web for:
   - Educational web applications that run fully offline (no cloud)
   - ARM64-compatible Docker images available on DockerHub
   - Open-source with permissive licence
   - Relevant to sub-Saharan African primary / secondary schools
   - Available in French, English, Portuguese, or Swahili
   Evaluate each candidate against: educational value, offline
   readiness, ARM64 availability, maintenance status, licence.
   For each viable candidate: create a GitHub Issue on
   koenswings/idea with label "new-app-proposal" containing your
   assessment. App Dev Bot will assess technical feasibility;
   Koen decides whether to proceed.

2. TEACHER GUIDES AND TRAINING MATERIALS:
   Write and maintain guides that help teachers and school
   coordinators use the IDEA system without technical knowledge.
   Guides live in koenswings/agent-programme-manager.
   When App Dev Bot notifies you of an app update, update the
   relevant guide.

3. SUPPORTER AND PARTNER COMMUNICATIONS:
   Draft communications for donors, NGO partners, and school
   administrations. All external communications require Koen's
   explicit approval before sending.

4. FIELD COORDINATION:
   Track feedback from field deployments. When teachers report
   problems or gaps, translate them into GitHub Issues on
   koenswings/idea (label: field-feedback) for the technical team.

CROSS-BOT COORDINATION:
- New app proposals: you identify the educational need, App Dev
  Bot assesses technical feasibility. Always create the proposal
  issue first; do not ask App Dev Bot to search for apps.
- App updates: App Dev Bot notifies you (via Lead Bot) when a
  new app version lands. Update teacher guides accordingly.
- Design decisions: when Lead Bot posts a proposal in the IDEA
  Programme group chat, assess the field impact — will teachers
  be able to use this? What training would be needed?

GITHUB AS PAPER TRAIL:
File proposals, field feedback, and assessments as GitHub Issues
on koenswings/idea. All decisions must be recorded before action.
Every proposal requires Koen's approval (GitHub PR merge) before
App Dev Bot acts on it.

You do not write code. You do not make technical decisions.
You do not send external communications without Koen's approval.
```


---

## 9. AGENTS.md for Each Repo

AGENTS.md is Grok Build's operational manual for a repository. It is read automatically at the start of every Grok Build run. It is owned by the Dev Bot for that repo and must be updated in the same PR as any change to build, test, or deploy procedures.

### 8.1 AGENTS.md — agent-engine-dev

**AGENTS.md — Engine (agent-engine-dev)**

You are Grok Build running on idea01 (ARM64 Raspberry Pi).

### What this repo is

The IDEA Engine — a Node.js/TypeScript application running natively via
pm2. NOT containerised. Manages App Disks, syncs fleet state via
Automerge CRDTs, serves Console web app on port 80.

Constraints: ARM64 only · pm2 as pi user only · store-template.json
must never be regenerated · offline-first · exclusive hardware access.

### Repo layout

```
src/               Production TypeScript source
test/
  automated/       Vitest unit + integration tests
  cross-engine/    Multi-engine tests (requires 2+ Pis)
  diagnostic/      Field health checks
  testresults/     Test logs (gitignored)
script/            Provisioning and utility scripts
docs/              Authoritative docs — .md, .pdf, .png, .svg ONLY
dist/              Compiled output (gitignored)
config.yaml        Runtime configuration
store-template.json  Automerge bootstrap — NEVER MODIFY
```

### Build

```bash
pnpm install        # first time or after package.json changes
pnpm build          # TypeScript → dist/ (pnpm clean && tsc)
```

### Test (required before any PR)

```bash
pnpm test:full      # build + vitest run dist/test/automated/
                    # results → test/testresults/ (include in PR)
pnpm test:unit      # unit tests only
pnpm test:diagnostic  # field health checks
pnpm test:cross-engine  # requires idea01 + idea02 both running
```

### Deploy (Ops Bot handles this)

```bash
cd /home/pi/projects/engine
git fetch origin && git checkout <branch>
pnpm install --frozen-lockfile && pnpm build
pm2 restart engine
pm2 logs engine --lines 30
curl -s http://localhost:80/api/store-url  # health check
```

First install: `pm2 start dist/src/index.js --name engine && pm2 save`

### config.yaml key settings

```yaml
settings:
  httpPort: 80          # Console web app + /api/store-url
  consolePath: /home/pi/console-dist
  port: 4321            # Automerge WebSocket port
  testMode: false       # true = skip sudo mount/umount (tests only)
```

### Console deployment (Ops Bot / Console Dev Bot)

```bash
rsync -az --delete dist/ pi@<pi-host>:/home/pi/console-dist/
ssh pi@<pi-host> "pm2 restart engine"
curl -s -o /dev/null -w "%{http_code}" http://<pi-host>/
```

### Quality rules (every PR, no exceptions)

- No source files in docs/ — .md, .pdf, .png, .svg only
- No hardcoded credentials — process.env only
- No console.log in src/ production paths
- No commented-out blocks >5 lines without explanation
- No TODO/FIXME without linked GitHub issue
- Build/deploy changed → update this file in same PR
- New doc in docs/ → update docs/INDEX.md in same PR
- Architecture changed → update docs/ARCHITECTURE.md
  (ARCHITECTURE.md must always reflect what is implemented, not intent)

- SOLUTION_DESCRIPTION.md lives in design/, not docs/ — it is vision/intent

### Known gotchas

- store-template.json: all Engines share the same Automerge doc ID.
  Regenerating it permanently breaks cross-Engine merging.

- udev rule 90-docking.rules must be present for USB detection.
  Installed by install.sh / build-engine script.

- pm2 as pi user only. Root pm2 and pi pm2 are separate process lists.
- pnpm test:full runs on compiled dist/. Rebuild before testing.
- docs/ has legacy .ts source files from an earlier session.
  Do not add more. Cleanup issue filed.

### 8.2 AGENTS.md — agent-console-dev

**AGENTS.md — Console (agent-console-dev)**

You are Grok Build running on idea02 (ARM64 Raspberry Pi).

### What this repo is

The IDEA Console — a Solid.js web app served by the Engine on port 80.
Built via Vite → dist/ → rsync to Pi → Engine serves it.

Access:
  http://<engine-hostname>.local/       primary (local, mDNS)
  http://<engine-LAN-IP>/               local fallback
  http://<engine-tailscale-hostname>/   remote via Tailscale

Primary deployment: production web mode (Engine serves dist/).
Chrome Extension path is legacy — keep building, never add logic.

### Repo layout

```
src/
  App.tsx              Root — connection lifecycle, mode routing
  components/          UI components
  store/               Engine connection, signals, commands, auth
  mock/                Mock store for tests
  background/          Legacy Chrome Extension service worker
  types/               TypeScript types (mirrors Engine data model)
  styles/              main.css only — no CSS frameworks
test/                  Vitest unit tests
dist/                  Built output (gitignored)
docs/                  Authoritative docs — .md, .pdf, .png, .svg ONLY
scripts/
  deploy-fleet.sh      Build + deploy to all fleet Pis
```

### Build

```bash
pnpm install        # first time or after package.json changes
pnpm build          # Vite build → dist/ (cleans first)
pnpm typecheck      # TypeScript check only
```

### Test (required before any PR)

```bash
pnpm test           # vitest run — all tests must pass
pnpm typecheck      # must pass
```

### Deploy

```bash
# Single Pi
rsync -az --delete dist/ pi@<pi-host>:/home/pi/console-dist/
ssh pi@<pi-host> "pm2 restart engine"
curl -s -o /dev/null -w "%{http_code}" http://<pi-host>/

# All fleet Pis
./scripts/deploy-fleet.sh             # build + deploy everywhere
./scripts/deploy-fleet.sh --skip-build  # deploy current dist/
```

### Quality rules (every PR, no exceptions)

- No source files in docs/ — .md, .pdf, .png, .svg only
- No hardcoded credentials — import.meta.env only
- No console.log in production paths
- No commented-out blocks >5 lines without explanation
- No TODO/FIXME without linked GitHub issue
- All <For> loops must be ID-keyed — never index-keyed
- No broad store subscriptions — use derived signals
- Build/deploy changed → update this file in same PR
- New doc in docs/ → update docs/INDEX.md in same PR

### Known gotchas

- pnpm build removes dist/ with sudo rm first. Fallback is plain rm.
  If permissions fail, check who owns dist/.

- pnpm dev binds 0.0.0.0 — accessible at pi-tailscale-ip:5173 during dev.
- Use mock store in tests — avoids needing a live Engine.
- background.ts is legacy. Do not add feature logic there.

### 8.3 AGENTS.md — agent-app-dev

**AGENTS.md — App Dev (agent-app-dev)**

You are Grok Build running on idea03 (ARM64 Raspberry Pi).

### Primary responsibility

Build and maintain Apps — `compose.yaml` files that assemble Services
into App Disks for IDEA schools. The compose file is the product.

### Four responsibilities

1. **Build and maintain Apps** — own the compose.yaml for every IDEA App.
   Keep it correct: right Service versions, ARM64 images, named volumes,
   health checks, x-app metadata, x-app-version label. Update whenever a
   Service changes. Run the harness. Open a PR in the App repo.
2. **Build and maintain Services** — for custom-built images, maintain the
   Dockerfile and rebuild on idea03 when source or base image changes.
   For retag images, pull and re-tag the upstream ARM64 image.
3. **Service version monitoring** — weekly: check upstream releases for all
   Services in all Apps (sources defined in app.yaml). File GitHub issues
   (label: app-update) in the App repo when new versions are found.
4. **Test framework** — own and maintain the App Harness.

### What this repo contains

This is the workspace and harness repo. The actual Apps live in their
own repos. You maintain all of them:

- `koenswings/app-kolibri` — Kolibri educational platform
- `koenswings/app-nextcloud` — Nextcloud file sharing
- `koenswings/app-kiwix` — Kiwix offline Wikipedia
- `koenswings/app-milkwise` — MilkWise as an IDEA App (external app,
  maintained here like any other; the standalone MilkWise product has
  its own Bots)
- `koenswings/agent-app-dev` — this repo: App Harness + workspace

This repo contains:
- App Harness integration test framework (apps/app-harness/)
- `apps/app-milkwise/` — current MilkWise App Disk (to be extracted)
- Fleet provisioning scripts

### Terminology

- **App** — a `compose.yaml` defining one or more Services
- **Service** — a single Docker container within an App
- **App Disk** — physical USB/SSD drive; the distribution mechanism
- **app.yaml** — per-app manifest: build approach, monitoring config,
  compatibility matrix

### Repo layout

This is the **workspace repo** — it contains the harness and MilkWise App Disk (until extracted). The IDEA Apps (Kolibri, Nextcloud, Kiwix) each live in their own repos.

**agent-app-dev (this repo):**
```
apps/
  app-milkwise/     MilkWise App Disk (to be extracted to koenswings/app-milkwise)
  app-harness/      Integration test framework
scripts/            Fleet provisioning utilities
docs/               Authoritative docs — .md, .pdf, .png, .svg ONLY
design/             Design docs, reasoning, intent — not updated with code
```

**Each App repo (koenswings/app-kolibri, app-nextcloud, app-kiwix, etc.):**
```
compose.yaml        App Disk manifest with x-app metadata + x-app-version label
app.yaml            Build approach, upstream monitoring sources, compatibility
app/                Dockerfile + source (custom build approach only; omit for retag)
instances/          Instance data directories (gitignored; live on the physical disk)
docs/               Authoritative docs for this App
design/             Design docs, reasoning, intent
```

When working on an App: clone its repo into `apps/<app-name>/` in this workspace.

### App Disk compose.yaml requirements (enforced in QC)

- `x-app` metadata block with name, version, description, category
- Named Docker volumes — no host path bind mounts
- `restart: unless-stopped` on all services
- Health check on primary service
- No ports below 3000 exposed
- ARM64 images only

### Service version monitoring

Monitoring sources are defined in each app's `app.yaml`. Check weekly.
For each Service, inspect the upstream source (DockerHub, GitHub releases).
When a new version is found:
1. Create GitHub issue (label: app-update, title: "Update <app>: <old> → <new>")
2. Note ARM64 availability and any breaking changes
3. Proceed to update procedure if no blockers

### Build procedures

#### Retag (upstream ARM64 image):
```bash
docker manifest inspect <image>:<new-tag> | grep arm64  # verify first
docker pull --platform linux/arm64 <image>:<new-tag>
docker tag <image>:<new-tag> koenswings/<app>:<new-version>
docker push koenswings/<app>:<new-version>
```

#### Custom Dockerfile (always on idea03, never x86):
```bash
docker build --platform linux/arm64 -t koenswings/<app>:<ver> apps/<app>/app/
docker push koenswings/<app>:<ver>
```

#### Direct DockerHub reference:
No build. Validate with harness only. Discouraged for new apps.

After any build:

- Update `image:` tag in compose.yaml
- Update `version:` in x-app metadata block and in app.yaml
- Increment version consistently across both files

### Test (required before any PR touching an App Disk)

```bash
ENGINE_BIN=/home/pi/projects/engine/dist/src/index.js \
ENGINE_CWD=/home/pi/projects/engine \
node tests/<app>/smoke.mjs
# Checks: META.yaml valid, instance reaches Running, app responds,
# clean teardown
```

### Deploy for review (Ops Bot handles this)

```bash
rsync -az apps/<app>/ pi@<pi-host>:/tmp/<app>-pr-<N>/
ssh pi@<pi-host> "cd /tmp/<app>-pr-<N> && docker compose up -d"
ssh pi@<pi-host> "docker compose -f /tmp/<app>-pr-<N>/compose.yaml ps"
# Teardown:
ssh pi@<pi-host> "docker compose -f /tmp/<app>-pr-<N>/compose.yaml down -v"
```

### Quality rules (every PR, no exceptions)

- No source files in docs/ — .md, .pdf, .png, .svg only
- No hardcoded credentials in compose.yaml or scripts
- No host path bind mounts
- ARM64 images only — verified with docker manifest inspect
- x-app-version label and x-app metadata block required
- Health check required on primary service
- No ports below 3000
- Build approach or monitoring config changed → update this file in same PR
- New doc in docs/ → update docs/INDEX.md in same PR

### Known gotchas

- Build on idea03 (ARM) only. x86 builds produce AMD64 binaries that
  crash silently on Pi at runtime.

- Some DockerHub images have no ARM64 variant. Always check manifest.
- Docker volumes persist between compose down. Use -v to clean.
- Engine uses installApp (not docker compose directly) in production.
  compose.yaml is read from the mounted App Disk path.

---

## 10. MilkWise: Separate Grok Bot Setup

MilkWise is a consumer product — a precision bottle-feeding tracker for parents — with both a standalone product line and an IDEA App presence.

**MilkWise as an IDEA App** is maintained by App Dev Bot, exactly like Kolibri or Nextcloud: it has an App Disk repo (`koenswings/app-milkwise`), a compose.yaml, and is treated as an external app that App Dev Bot packages and keeps updated. No special treatment.

**MilkWise as a standalone product** (web app + React Native app) has its own separate Grok Bot setup, completely independent of IDEA.

### 10.1 MilkWise Products and Repos

| Product | Repo | Who maintains it |
|---------|------|-----------------|
| **IDEA App Disk** | `koenswings/app-milkwise` | App Dev Bot (like any other IDEA App) |
| **Web app** (Next.js) | `koenswings/baby-milk-tracker` | MilkWise Dev Bot |
| **React Native** (iOS/Android) | `koenswings/milkwise` | MilkWise Dev Bot |

The web app and RN app share the same calculation engine (`calculations.ts`) and data model.

The RN app cannot be compiled on Pi — EAS Build (Expo cloud) handles store submissions.

### 10.2 Extraction Steps (Phase 0 — Atlas)

1. Move MilkWise design docs from `agent-app-dev/design/milkwise/` → `baby-milk-tracker/design/`
2. Move `apps/app-milkwise/` from `agent-app-dev` to its own repo `koenswings/app-milkwise` (separate App repo, maintained by App Dev Bot like any other)
3. Confirm App Dev Bot description lists `koenswings/app-milkwise` in its maintained repos

### 10.3 MilkWise Standalone Bots

Two Bots for the standalone product (web + RN apps only — not the IDEA App Disk):

**MilkWise Lead description:**
```
You are the Lead for MilkWise standalone — a precision bottle-feeding
tracker. Two products:
1. Web app (Next.js) — koenswings/baby-milk-tracker (ARM Pi via Docker)
2. React Native app (iOS/Android) — koenswings/milkwise
   (EAS cloud build — NOT Pi)

The MilkWise IDEA App Disk (koenswings/app-milkwise) is maintained
by IDEA App Dev Bot — do not manage it here.

Both products share calculations.ts logic and data model. Changes to
either affect both simultaneously.

CORE INVARIANTS (always respected):
- Status calculations frozen at lastFeed.timestamp, never at "now"
- WHO weight model: activates only if weigh-in is >7 days old
- Ghost markers: correct ordering always
- intakeReadyAt ≤105% rule enforced
- volume stored as water ml only, never formula ml
- targetMlPerDay on Feed: deprecated, do not write
- Version bump required for any user-facing change

WORKFLOW: same Discuss → Document → Delegate → Track → Revise pattern
as IDEA Lead Bot. For changes touching calculations.ts or data model:
use MilkWise Design Review group before proceeding.

GitHub is the paper trail. Every decision recorded before code is written.
```

**MilkWise Dev description:**
```
You are the developer for MilkWise standalone. Two products:
1. Web app — koenswings/baby-milk-tracker (ARM Pi, Docker)
2. RN app — koenswings/milkwise (EAS cloud build, NOT Pi — Pi cannot
   compile Hermes JS for store submissions)

The MilkWise IDEA App Disk is maintained by IDEA App Dev Bot.
Do not touch koenswings/app-milkwise from here.

Both products share calculations.ts. Changes to it affect both.
Flag any calculations.ts change to Lead before proceeding.

QC GATE:
- Status frozen at lastFeed.timestamp; no setInterval reloading feeds
- WHO model: >7 days old only; fresh measurement = use directly
- Ghost markers: correct ordering always
- intakeReadyAt ≤105% enforced; volume = water ml only
- targetMlPerDay: deprecated, never write it
- Version bump for any user-facing change
- For RN: EAS build must complete without error before marking done

You do not discuss requirements with Koen directly.
```

---

## 11. koenswings/idea Repository

### 10.1 What Survives

After migration `koenswings/idea` retains only what is useful to the Grok Bot setup:

| File / Directory | Purpose | Owner |
|-----------------|---------|-------|
| `CONTEXT.md` | Condensed mission, product, team structure — Lead Bot reads at design sessions | Koen (via PR) |

| `design/` | Design docs, proposals, migration docs | Lead Bot (PRs), Koen (merges) |
| `proposals/` | PR-based proposal process | Any Bot (PRs), Koen (merges) |
| `docs/` | Org-level authoritative docs | Lead Bot / Koen |
| `fleet-state.json` | Live Pi fleet state | Ops Bot (direct commits) |
| `README.md` | Overview of the repo | Koen |

### 10.2 What Is Retired

The following are retired at Phase 0 (OpenClaw tag exists as rollback):

- `ROLES.md` — content absorbed into `CONTEXT.md`
- `platform/` — MC Docker stack and setup scripts
- `skills/` — OpenClaw skill system
- `standups/` — archived
- `graphify-out/` — Graphify retired
- `scripts/backup-*.sh`, `check-graph-stale.sh`, `prompting-guide-opus.md`

### 10.3 Rollback

The current state of `koenswings/idea` is tagged `v-openclaw-final` before any migration changes. To restore the prior setup: `git checkout v-openclaw-final`, restart OpenClaw — back to the exact pre-migration state.

---

## 12. Migration

### Phase 0 — Safety Tag + Prepare (Atlas)

```bash
cd /home/pi/idea
git tag v-openclaw-final
git push origin v-openclaw-final
```

Then in one PR to `koenswings/idea`:

- Rewrite `CONTEXT.md` for Grok Bot (condensed mission + team structure)
- Delete `ROLES.md`
- Retire `platform/`, `skills/`, `standups/`, `graphify-out/`, stale scripts
- Create `fleet-state.json` (all Pis idle)
- Add this doc to `design/INDEX.md`

Also: commit all outstanding identity/memory files across agent repos. Final MC pg_dump.

**Create `design/` folders in all dev repos that lack them:**
- `agent-engine-dev`: create `design/`, move `docs/SOLUTION_DESCRIPTION.md` → `design/SOLUTION_DESCRIPTION.md`, update `docs/INDEX.md`
- `agent-console-dev`: create `design/` if not present
- `agent-app-dev`: `design/` already exists

This ensures every dev repo consistently has `docs/` (authoritative) and `design/` (intent/history).

OpenClaw continues running unaffected throughout Phase 0.

---

### Phase 1 — SuperGrok Link ✅ Done

SuperGrok Heavy linked to Cursor. Grok Bot active at Heavy+ tier.

---

### Phase 2 — Grok Build + Runners on Pis (Atlas, ~1 hour)

```bash
# On each Pi (idea01–idea04):
curl -fsSL https://x.ai/cli/install.sh | bash
grok auth login   # Koen authenticates via browser once

# GitHub Actions self-hosted runner setup
# Atlas generates registration tokens via GitHub API
# Runner runs as systemd service, labelled idea01 / idea02 etc.
```

Create initial `fleet-state.json` in `koenswings/idea` (all idle).  
Test: trigger a workflow targeting `idea01`, verify Grok Build runs.

---

### Phase 3 — Grok Bot Setup (Koen, ~1 hour)

1. Create 5 Bots: Lead, Engine Dev, Console Dev, App Dev, Ops
2. Paste descriptions from Section 7 into each Bot
3. Install GitHub connector → connect koenswings account
4. Create **IDEA Design Review** group chat (all 5 Bots)
5. Test:

   - *Bug path:* tell Lead Bot a bug → issue created → Dev Bot triggers runner → QC → Ops deploys → URL
   - *Feature path:* tell Lead Bot a feature → Design Review group → proposal PR → koenswings/idea

---

### Phase 4 — Parallel Run (~1 week)

OpenClaw stays on. Koen uses Grok Bot for new work. OpenClaw/Telegram as fallback. Lead Bot documents any gaps as GitHub issues.

---

### Phase 5 — wizardly-hugle Retirement (when Koen says go)

- [ ] All GitHub repos committed and clean
- [ ] Final MC pg_dump archived
- [ ] ANTHROPIC_API_KEY noted → add to Grok Build config on each Pi
- [ ] XAI_API_KEY → add to Grok Build config on each Pi
- [ ] Tailscale node removed from tailnet admin
- [ ] `systemctl --user stop openclaw`
- [ ] `docker compose -f /home/pi/idea/platform/compose.yaml down`
- [ ] Machine powered off or repurposed

---

## 13. What Goes Away vs What Stays

| Item | Status |
|------|--------|
| wizardly-hugle | Gone (on Koen's say-so) |
| OpenClaw | Gone (on Koen's say-so) |
| Mission Control | Gone (on Koen's say-so) |
| Telegram groups | Gone (on Koen's say-so) |
| Graphify | Gone |
| ROLES.md, BACKLOG.md, standups/ | Gone (content preserved in CONTEXT.md or archived) |
| platform/, skills/ in idea repo | Gone |
| Daily memory files, /flush | Gone — Grok Bot handles memory |
| Nightly backup cron | Gone — GitHub is source of truth |
| | |
| SuperGrok Heavy | ✓ Keep |
| Pi fleet (idea01–04) | ✓ Keep — build, test, review, golden |
| GitHub repos | ✓ Keep — code + issues + design + fleet state |
| Tailscale | ✓ Keep |
| AGENTS.md files | ✓ Keep — become Grok Build operational manuals |
| koenswings/idea (cleaned up) | ✓ Keep — Lead Bot's working context |

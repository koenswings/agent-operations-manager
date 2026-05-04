# Design: Graphify Integration into IDEA Platform

**Status:** Draft — Implementation Plan Added  
**Author:** Atlas  
**Date:** 2026-05-03  
**Related:** https://github.com/safishamsi/graphify  
**Additional reading:** https://medium.com/agentic-builders/how-to-use-graphify-turn-any-folder-into-a-knowledge-graph-d51b38eb60b6

---

## 1. What is Graphify?

Graphify is a Python tool (PyPI: `graphifyy`) that ingests a folder of code, docs, PDFs, images, and video, and builds a **persistent, queryable knowledge graph** of concepts and relationships.

Output:
- `GRAPH_REPORT.md` — human-readable audit: god nodes, surprising connections, suggested queries
- `graph.json` — machine-queryable graph, persistent across sessions
- `graph.html` — interactive visual graph (click nodes, search, filter by community)
- `obsidian/` — optional: ready-to-use Obsidian vault with backlinks (via `--obsidian` flag)

**The core mechanism — compressed subgraph serving:** Graphify compresses the codebase into a structured graph that agents query instead of reading raw files. On platforms that support it (Claude Code, Codex), a PreToolUse hook fires before every file-read and serves a focused ~300-token subgraph on demand. **OpenClaw does not support this hook.** For OpenClaw (and also Aider, Factory Droid, Trae), Graphify falls back to session-start injection via AGENTS.md — `GRAPH_REPORT.md` is included at the top of every session automatically via a directive written by `graphify claw install`. This is a meaningful capability difference: Claude Code agents navigate the graph dynamically mid-session; OpenClaw agents get structural context once at session start.

**Graphify is a skill, not a standalone orchestrator.** The coding assistant (OpenClaw, Claude Code, Codex, etc.) is the runtime. Graphify dispatches extraction subagents in parallel, validates output against a schema, and merges it into the graph.

**Benchmarked token reduction:** 71.5x fewer tokens per query vs reading raw files (52-file mixed code+docs+images corpus). Token savings compound with corpus size.

Every edge carries a provenance tag — `EXTRACTED` (AST-derived, confidence 1.0), `INFERRED` (LLM-derived, with confidence score), or `AMBIGUOUS`. You always know what was found vs guessed.

---

## 2. Why This Is Relevant to IDEA

### 2.1 The cross-agent coordination problem

IDEA's five agents cannot talk to each other directly. Cross-agent coordination happens through shared files — CONTEXT.md, ARCHITECTURE.md, design docs. This means every agent reading a foreign codebase (Axle reading Console types, Pixel reading Engine events, Atlas reviewing a PR across both) must either:

- Read many files cold, burning tokens and time, or
- Rely on ARCHITECTURE.md being current (it lags)

A committed, auto-updating knowledge graph in the `idea/` repo gives every agent an instant structural map of the whole platform — without any agent needing to maintain it manually. GRAPH_REPORT.md is injected at session start via AGENTS.md, giving every agent structural context from the first turn of every session.

### 2.2 Token cost on the Pi

OpenClaw runs natively on a Raspberry Pi 500+. Token cost is a real constraint. The 71.5x reduction matters most during deep cross-repo work — exactly the sessions that currently burn the most tokens.

### 2.3 CONTEXT.md is already a hand-maintained graph

CONTEXT.md is a hand-written knowledge summary — updated manually, prone to lag. A live graph automates the structural layer, letting CONTEXT.md stay focused on mission and principles while the graph owns architecture and code relationships.

### 2.4 Cross-domain connections surface automatically

The graph would reveal relationships that currently require manual effort to document:
- Engine Automerge sync logic ↔ Console state management
- App Disk metadata schema ↔ container lifecycle
- `compose.yaml` service names ↔ Engine/Console config keys
- Design docs in `design/` ↔ the code that implements them

Marco and Beacon can also use the graph as a non-technical entry point to "what does the system actually do" — grounding comms, teacher guides, and website copy in real architecture without reading source.

---

## 3. Architectural Decisions

### 3.1 Single graph from the `idea/` root (not per-repo)

**Question:** Why not build one graph from the idea root folder to leverage cross-repo knowledge for all agents?

**Answer: Yes — this is the right approach.**

The `idea/` root already contains all agent repos under `agents/` (mounted as submodules/subdirs). Running `/graphify .` from the idea root captures everything in a single pass:

- All TypeScript/JS source (Engine, Console) — free, AST via tree-sitter
- All shared docs (CONTEXT.md, ARCHITECTURE.md, ROLES.md, PROCESS.md, design docs) — one-time API cost, then cached
- All agent memory and identity files — can be excluded via `.graphifyignore` (sensitive, not useful for code navigation)

Cross-repo edges (Engine ↔ Console, code ↔ design docs, compose.yaml ↔ Engine config) emerge naturally in one pass. No manual merging. No sync step. One `graphify-out/` committed to the `idea/` repo, readable by every agent.

**Comparison vs per-repo approach:**

| | Per-repo graphs | Single root graph |
|---|---|---|
| Cross-repo edges | Requires manual merge step | Native, automatic |
| Maintenance | 2–5 graphs to keep in sync | One graph |
| Surprising connections | Limited to intra-repo | Full platform-wide |
| Setup complexity | Higher | Lower |

**Decision: Build one graph from `idea/` root. Per-repo graphs are not needed.**

`.graphifyignore` at the idea root:
```
# exclude agent memory/identity (sensitive, not useful for code nav)
agents/*/memory/
agents/*/MEMORY.md
agents/*/.git/

# exclude build artifacts
**/node_modules/
**/dist/
**/.next/
**/target/
```

### 3.2 Session-start injection via AGENTS.md (OpenClaw's only mechanism)

**Correction from earlier draft:** A previous version of this doc argued for "hook only" to save tokens. That was wrong — the PreToolUse hook is not available on OpenClaw at all. OpenClaw agents have exactly one mechanism: session-start injection via AGENTS.md.

**What `graphify claw install` does on OpenClaw:**  
Writes a directive into AGENTS.md (at the project root) telling every OpenClaw agent to read `graphify-out/GRAPH_REPORT.md` at the start of each session. That's it. There is no mid-session hook, no dynamic subgraph serving, no on-demand query mechanism during normal operation.

**What this means in practice:**

| Mechanism | Claude Code / Codex | OpenClaw (IDEA) |
|-----------|--------------------|-----------------|
| PreToolUse hook (mid-session, dynamic) | ✅ Available | ❌ Not available |
| Session-start injection via AGENTS.md | Optional supplement | ✅ Only option |
| MCP server (opt-in, deep sessions) | ✅ Available | ✅ Available |

**The session-start model is still valuable — it's just different:**  
Every agent starts every session with the structural map of the entire platform already in context. Conceptual queries ("how does sync work?"), architecture questions, cross-repo reasoning — all of these get the graph without the agent having to explicitly call for it. The limitation vs Claude Code is that the agent doesn't get a fresh focused subgraph on each tool call; it works from the session-start snapshot.

**Token cost of session-start injection:**  
GRAPH_REPORT.md will be ~500–1,000 tokens. This is paid once per session (injected at the top, cached for subsequent turns). For sessions that do cross-codebase work, this cost pays for itself quickly. For short single-file sessions (e.g. a cron heartbeat), it's modest overhead. On balance, the structural benefit outweighs the overhead for IDEA's agent workloads.

**Handling long-running sessions:**  
Since nightly reboots are disabled on the Pi, agent sessions can run for days or weeks. GRAPH_REPORT.md is injected once at session start — meaning a long-running session works from a snapshot that may predate several merged PRs. Mitigation: agents should explicitly re-read `graphify-out/GRAPH_REPORT.md` at the start of any significant cross-codebase task. This is a standing instruction added to each agent's AGENTS.md (see §8, Phase 2).

**MCP server (optional, for deep work sessions):**  
Graphify ships an MCP server that exposes the full `graph.json` as queryable tools. This is available to OpenClaw agents as an opt-in for sessions that need dynamic subgraph queries — e.g. a deep refactor session where an agent wants to ask "what are all the callers of this function across both repos?" The MCP server is not part of the base integration; it's an upgrade path if the session-start model proves insufficient.

### 3.3 Accessing the graph from a remote Mac

**Question:** Be able to navigate the graph from a remote Mac — both the HTML and the Obsidian vault.

Two deliverables, two different approaches:

#### HTML interactive viewer

Serve `graphify-out/graph.html` as a static file via a lightweight HTTP server on the Pi, accessible over Tailscale.

```bash
# systemd user service on pi: graphify-serve.service
python3 -m http.server 8083 --directory /home/pi/idea/graphify-out
```

Accessible from Mac at: `http://idea.tail2d60.ts.net:8083/graph.html`

This should be a named systemd user service (like OpenClaw), so it survives reboots. Port 8083 is suggested (8080/8081/8082 likely already used); confirm no conflicts. No auth needed — Tailscale handles access control.

#### Obsidian vault

Built via `graphify . --obsidian` (generates `graphify-out/obsidian/`). Since `graphify-out/` is committed to the `idea/` repo on GitHub:

1. Koen clones/pulls `koenswings/idea` on his Mac
2. Opens `graphify-out/obsidian/` as an Obsidian vault
3. When the graph updates (post-commit hook → PR merge → `git pull` on Mac), vault updates automatically

No additional infrastructure needed. A `git pull` is the refresh mechanism. If auto-sync is preferred, Obsidian Sync or a simple cron `git pull` on the Mac could automate it — but manual pull is likely fine given graph rebuilds are infrequent.

---

## 4. The Extraction Pipeline

Understanding the three-pass pipeline matters for knowing what costs what and what runs where:

**Pass 1 — Deterministic AST parsing (local, free)**  
Tree-sitter parses all code files locally. No API calls. Extracts classes, functions, imports, call graphs, docstrings, rationale comments (`# NOTE:`, `# WHY:`, `# HACK:`). Every edge tagged `EXTRACTED`, confidence 1.0.

**Pass 2 — Local transcription (local, free)**  
Audio/video files transcribed on-device via faster-whisper. SHA256-cached. Never leaves the machine. Not currently relevant for IDEA but zero cost if it ever is.

**Pass 3 — Parallel LLM extraction (API cost, once per file)**  
For docs, PDFs, images, transcripts — coding assistant dispatches subagents in parallel to extract concepts and relationships. Schema-validated before merging. Edges tagged `INFERRED` with confidence score. SHA256-cached — only changed files reprocess on subsequent runs.

**Cost profile for IDEA:**

| Content | Pass | Cost |
|---|---|---|
| TypeScript/JS source (Engine, Console) | Pass 1 (AST) | Free, local |
| SQL schemas | Pass 1 | Free, local |
| YAML/JSON config files | Pass 1/3 | Minimal |
| CONTEXT.md, ARCHITECTURE.md, ROLES.md, PROCESS.md | Pass 3 (LLM) | One-time, then cached |
| Design docs in `design/` | Pass 3 (LLM) | One-time, then cached |
| Agent memory files | Excluded via `.graphifyignore` | Zero |

Bottom line: the first build has a modest API cost for the docs layer. Every subsequent rebuild (post-commit, code changes) costs nothing for the code and only re-processes changed docs.

---

## 5. Keeping the Graph Fresh

### 5.1 Rebuilding the graph: GitHub Actions (not local git hooks)

IDEA's workflow is PR-based: all changes go via PR, Koen merges, main advances on GitHub. A local git hook (`graphify hook install`) fires on the committing machine — in IDEA's case that's a Pi running an Engine, not a Mac laptop. Running a graphify build (Pass 3 LLM extraction) on commit would hit the same Pi that's running an active Engine session, competing for RAM and API budget mid-session. That's the wrong place for it.

GitHub Actions is the correct trigger: when a PR merges to main, **GitHub spins up a fresh Ubuntu VM in Microsoft's Azure cloud** — completely isolated from the Pi, your network, and whatever agents are doing. The VM installs graphify, builds the graph (API calls go directly from Azure to Anthropic), commits `graphify-out/` back to main, and is destroyed. The Pi is never involved in the build. The full flow:

```
Agent pushes branch on Pi → PR merged on GitHub
  → GitHub launches Ubuntu VM (Azure cloud)
  → VM: checkout repo, pip install graphifyy
  → VM: /graphify . (Pass 3 API calls go Azure → Anthropic)
  → VM: git commit graphify-out/ → push to main
  → VM destroyed
  → Pi (up to 30 min later): git pull → graph available
```

```yaml
# idea/.github/workflows/graphify.yml
name: Rebuild knowledge graph
on:
  push:
    branches: [main]
jobs:
  graphify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: pip install graphifyy
      - run: graphify . --update --no-viz
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      - name: Commit updated graph
        run: |
          git config user.name "graphify-bot"
          git config user.email "graphify@idea"
          git add graphify-out/
          git diff --staged --quiet || git commit -m "chore: rebuild knowledge graph [skip ci]"
          git push
```

`[skip ci]` in the commit message prevents the workflow from triggering itself.

Code-only merges (TypeScript changes) complete in seconds — AST rebuild, no API cost. Doc changes trigger Pass 3 for only the changed files — one-time cost per changed doc.

### 5.2 Getting the updated graph onto the Pi: isolated sparse clone + cron

The naïve approach — `cd /home/pi/idea && git pull` on cron — has a real conflict risk: agents work in the `/home/pi/idea` working directory (feature branches, staged changes, ongoing commits). A cron `git pull` hitting the same working tree while an agent is mid-operation can corrupt state or fail silently.

**The fix: a dedicated read-only sparse clone just for the graph output.**

This clone lives at a separate path (`/home/pi/graphify-cache`), tracks only `graphify-out/`, and is never touched by agent work:

```bash
# One-time setup on Pi (run once during Phase 3):
git clone --no-checkout --depth 1 --filter=blob:none \
  https://github.com/koenswings/idea.git /home/pi/graphify-cache
cd /home/pi/graphify-cache
git sparse-checkout set graphify-out/
git checkout main
```

```bash
# cron on Pi (pi user), every 30 minutes:
*/30 * * * * cd /home/pi/graphify-cache && git pull --quiet 2>>/home/pi/idea/logs/graphify-pull.log
```

Agents read from `/home/pi/graphify-cache/graphify-out/GRAPH_REPORT.md` — a path that is:
- Never written by agents (agents never work in `/home/pi/graphify-cache`)
- Only ever updated by the cron pull
- Sparse — only downloads `graphify-out/`, not the full repo history

No conflicts possible. The idea working directory and the graph cache are completely separate paths with no shared git state.

Lag between a merge landing on GitHub and the Pi having the updated graph: up to 30 minutes. Acceptable for a knowledge graph that reflects architectural state.

### 5.3 How agent sessions pick up the update

This is where an important constraint applies: **OpenClaw sessions are long-running.** After disabling the nightly reboot, sessions can run for days or weeks. GRAPH_REPORT.md is injected at session start — meaning a long-running session will work from a snapshot that may be many merged PRs out of date.

Session-start injection is therefore not sufficient on its own. The complementary mechanism is: **agents explicitly re-read GRAPH_REPORT.md at the start of significant work tasks.**

This should be a standing instruction in each agent's AGENTS.md:

```
Before starting any cross-codebase task, read:
  /home/pi/graphify-cache/graphify-out/GRAPH_REPORT.md
This is the current knowledge graph of the full IDEA platform.
Do not read this on every session start — only before cross-codebase work.
```

This makes the graph pull deliberate and current, rather than relying on it being fresh from session start.

---

## 6. Expected Benefits by Agent

| Agent | Benefit |
|---|---|
| **Atlas** | Cross-codebase PR review with structural context; surfaces design doc ↔ implementation drift |
| **Axle** | Navigates Automerge complexity faster; Engine↔Console API boundaries visible without reading Console source |
| **Pixel** | Engine event/state types navigable without reading Engine source; SolidJS reactivity graph |
| **Marco** | Non-technical map of "what the system actually does" for teacher guides, fundraising copy |
| **Beacon** | Same as Marco — website copy grounded in real architecture |
| **Koen** | HTML visual + Obsidian vault for architecture review, onboarding new contributors, understanding system state |

---

## 7. Risks and Limitations

**Graph quality scales with corpus size.** Engine and Console are already large enough to benefit. Will compound as the codebase grows.

**First build has a one-time API cost** for docs/markdown. Subsequent runs only reprocess changed files (SHA256 cache). Estimate: modest — the codebase is primarily TypeScript (free) and a handful of markdown docs.

**Graph currency in long-running sessions.** Sessions can run for days or weeks without restarting. Session-start injection of GRAPH_REPORT.md is stale for any session that started before the last merge. Mitigated by agents explicitly re-reading the file at task start (see §5.3). The graph is structural context, not ground truth — agents should verify against source for critical decisions.

**Build should run on a dev machine or CI, not the Pi.** The Python extraction CLI is compute-light but shouldn't run on the Pi during active sessions. Best practice: run `/graphify .` on a dev machine after significant changes, commit the output, let the Pi pull it. The git hook can also run on the dev machine.

**Tool maturity.** Graphify launched April 3, 2026 — approximately 4 weeks old at time of writing. ~130 commits, API and output formats may still shift. Not a battle-tested tool for mission-critical pipelines. MIT license, active development. Treat as a pilot, not a permanent dependency until it stabilises.

**Sequential processing on OpenClaw.** Unlike Claude Code (parallel subagents for Pass 3), OpenClaw processes docs sequentially during graph building. First build on a large doc corpus will be slower. Subsequent incremental builds are unaffected.

**No PreToolUse hook on OpenClaw — session-start injection only.** The mid-session dynamic subgraph mechanism is a Claude Code / Codex feature. On OpenClaw, Graphify uses session-start injection via AGENTS.md exclusively. Agents work from a snapshot injected at session start rather than receiving focused subgraphs per query. This is a real capability gap vs Claude Code, but structural context at session start is still meaningfully better than no graph at all. The MCP server is available as an opt-in upgrade for sessions that need dynamic queries (see §3.2).

**`graph.json` size.** Can grow on large codebases. Recommend committing `graph.json`, `GRAPH_REPORT.md`, and `obsidian/`. Gitignore `manifest.json`, `cost.json`, and optionally `cache/` (large, safe to regenerate).

---

## 8. Implementation Plan

> **Design principle:** Every change below is additive. Nothing existing is modified or deleted in Phase 1–3. Rollback at any phase leaves the platform in exactly the state it was before that phase. The fallback for each phase is listed explicitly.

---

### Phase 1 — Build and validate the graph (off-Pi, no Pi changes)

**Who:** Koen on his Mac (or Atlas can drive via shell if given access)  
**Pi changes:** None  
**Rollback:** Delete the local folder — nothing was committed  

#### Steps

```bash
# 1. Install graphify on your Mac (Python 3.10+)
pip install graphifyy

# 2. Clone the idea repo with all submodules (if not already local)
git clone --recurse-submodules https://github.com/koenswings/idea.git ~/idea-local
cd ~/idea-local

# 3. Create .graphifyignore at the repo root
cat > .graphifyignore << 'EOF'
# Agent memory/identity — sensitive, not useful for code nav
agents/*/memory/
agents/*/MEMORY.md
agents/*/.git/

# Build artifacts
**/node_modules/
**/dist/
**/.next/
**/target/

# Graphify output (don't recurse into its own output)
graphify-out/
EOF

# 4. Run the first build (inspect output before committing anything)
/graphify .
# This opens graph.html in browser automatically
# GRAPH_REPORT.md and graph.json land in graphify-out/
```

**Gate:** Review `graph.html` and `GRAPH_REPORT.md`. If the graph looks wrong or useless, stop here — nothing has touched the Pi or any repo. If it's useful, proceed.

```bash
# 5. Add graphify-out/ to .gitignore EXCEPT the files we want to commit
# (manifest.json, cost.json, cache/ are large/ephemeral — don't commit)
cat >> .gitignore << 'EOF'
graphify-out/manifest.json
graphify-out/cost.json
graphify-out/cache/
EOF

# 6. Commit .graphifyignore, updated .gitignore, and graphify-out/ to a branch
git checkout -b feature/graphify-graph
git add .graphifyignore .gitignore graphify-out/
git commit -m "feat: add graphify knowledge graph"
git push origin feature/graphify-graph
# → Open PR for review. Merge when happy.
```

**What exists after Phase 1:**
- `graphify-out/GRAPH_REPORT.md` — in repo, readable by agents
- `graphify-out/graph.json` — in repo, machine-queryable
- `graphify-out/graph.html` — in repo, visual viewer
- `graphify-out/obsidian/` — in repo, Obsidian vault
- `.graphifyignore` — in repo root
- `graphify-out/manifest.json`, `cost.json`, `cache/` — gitignored

---

### Phase 2 — Agent integration (AGENTS.md changes only)

**Who:** Atlas  
**Pi changes:** One line added to each agent's `AGENTS.md`. No services. No system config.  
**Rollback:** Remove the added lines from each AGENTS.md. Agents revert to not knowing about the graph — exactly current behaviour.

#### What changes and where

For **each agent** (`agents/agent-*/AGENTS.md`), add one instruction block. The exact wording:

```markdown
## Knowledge Graph

Before starting any cross-codebase task (reading files in another agent's repo,
reviewing a PR that spans Engine + Console, writing architecture docs), read:

  /home/pi/idea/graphify-out/GRAPH_REPORT.md

This gives you current structural context without reading raw source files.
Do not read this file on every session start — only when doing cross-codebase work.
```

**Files modified:**
- `agents/agent-operations-manager/AGENTS.md`
- `agents/agent-engine-dev/AGENTS.md`
- `agents/agent-console-dev/AGENTS.md`
- `agents/agent-programme-manager/AGENTS.md`
- `agents/agent-site-dev/AGENTS.md`

Done via PR to each agent's repo (same pattern as any other AGENTS.md update).

**What does NOT change:**
- No systemd services
- No `~/.openclaw/openclaw.json`
- No Pi cron
- No session-start injection (we're explicitly NOT running `graphify claw install`)
- CONTEXT.md and ARCHITECTURE.md remain unchanged — we do NOT remove manually-maintained docs yet

**Gate:** Run one test session with Axle. Ask about a Console concept without giving Console files. Does the answer reference the graph or read Console source cold? If the graph adds value, proceed. If not, remove the AGENTS.md line — zero other cleanup needed.

---

### Phase 3 — Automation: GitHub Actions + Pi cron

**Who:** Atlas (GitHub Actions workflow) + root crontab on Pi (one new line)  
**Pi changes:** One crontab entry for the `pi` user  
**Rollback:** Remove the crontab entry. Remove the GitHub Actions workflow file via PR. Pi reverts to not auto-pulling; graph stays static (last committed version — still useful, just not auto-updating).

#### 3a. GitHub Actions workflow

New file: `idea/.github/workflows/graphify.yml`

```yaml
name: Rebuild knowledge graph
on:
  push:
    branches: [main]
    paths-ignore:
      - 'graphify-out/**'    # don't trigger on graph's own commits
      - '**.md'              # optional: skip doc-only pushes to save API cost
                             # remove this line to include doc changes
jobs:
  graphify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive
          token: ${{ secrets.GITHUB_TOKEN }}
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: pip install graphifyy
      - run: graphify . --update --no-viz
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      - name: Commit updated graph
        run: |
          git config user.name "graphify-bot"
          git config user.email "graphify@idea"
          git add graphify-out/
          git diff --staged --quiet || git commit -m "chore: rebuild knowledge graph [skip ci]"
          git push
```

**Required:** Add `ANTHROPIC_API_KEY` to the `idea` repo's GitHub secrets (Settings → Secrets → Actions).

#### 3b. Pi: isolated sparse clone + cron

**Why not `git pull` on the idea working directory?**  
Agents work in `/home/pi/idea` — creating branches, staging changes, mid-commit. A cron `git pull` on that same directory while an agent is operating is a real conflict risk: git file locking, HEAD drift, or silent failure. The fix is a **dedicated read-only sparse clone** that only contains `graphify-out/` and is never touched by agent work.

**One-time setup on Pi (run once during Phase 3 rollout):**
```bash
git clone --no-checkout --depth 1 --filter=blob:none \
  https://github.com/koenswings/idea.git /home/pi/graphify-cache
cd /home/pi/graphify-cache
git sparse-checkout set graphify-out/
git checkout main
```

**Crontab entry** (add via `crontab -e` as `pi` user):
```
# Pull updated graphify graph into isolated cache every 30 minutes
*/30 * * * * cd /home/pi/graphify-cache && git pull --quiet 2>>/home/pi/idea/logs/graphify-pull.log
```

**What changes on the Pi (complete list for Phase 3):**

| Item | Location | Change |
|------|----------|--------|
| Sparse clone | `/home/pi/graphify-cache/` | New directory (contains only `graphify-out/`) |
| Crontab entry | `pi` user crontab | +1 line |
| Pull log | `/home/pi/idea/logs/graphify-pull.log` | New file (auto-created, append-only) |

The `/home/pi/idea` working directory is **never touched** by this mechanism. No conflict with agent work possible.

**Rollback Phase 3:**
```bash
# On Pi:
crontab -e  # delete the graphify-pull line
rm -rf /home/pi/graphify-cache
rm /home/pi/idea/logs/graphify-pull.log  # optional

# On GitHub: delete the workflow file via PR
# Graph stays in idea repo at last committed state — not a problem
```

---

### Phase 4 — Mac access (Pi HTTP service + Obsidian vault)

**Who:** Atlas  
**Pi changes:** One new systemd user service (`graphify-serve.service`)  
**Rollback:** `systemctl --user disable --now graphify-serve` + delete the unit file. Port 8083 is released. Nothing else on the Pi changes.

#### 4a. HTML viewer — systemd service on Pi

New file: `/home/pi/.config/systemd/user/graphify-serve.service`

```ini
[Unit]
Description=Graphify knowledge graph HTTP viewer
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 -m http.server 8083 --directory /home/pi/graphify-cache/graphify-out
Restart=on-failure
RestartSec=10

[Install]
WantedBy=default.target
```

```bash
# Enable and start
systemctl --user daemon-reload
systemctl --user enable --now graphify-serve

# Verify
curl -I http://localhost:8083/graph.html
```

Accessible from Mac over Tailscale at: **`http://idea.tail2d60.ts.net:8083/graph.html`**

**Port conflict check:** Before deploying, confirm port 8083 is free:
```bash
ss -tlnp | grep 8083
```
If something is there, use 8084 or 8090 instead (update both the service file and the URL).

#### 4b. Obsidian vault — Mac-side only

No Pi changes. The vault is in `graphify-out/obsidian/` which is already in the repo (Phase 1). Koen:

```bash
# On Mac: clone or pull the idea repo
git clone --recurse-submodules https://github.com/koenswings/idea.git ~/idea
# or, if already cloned:
cd ~/idea && git pull

# Open Obsidian → "Open folder as vault" → select ~/idea/graphify-out/obsidian/
```

To refresh the vault after a graph rebuild: `git pull` in `~/idea` on Mac.

**What changes on the Pi (complete list for Phase 4):**

| Item | Location | Change |
|------|----------|--------|
| systemd unit | `/home/pi/.config/systemd/user/graphify-serve.service` | New file |
| systemd symlink | Created by `systemctl enable` | Auto-created |
| Open port | TCP 8083 | New |

**Rollback Phase 4:**
```bash
systemctl --user disable --now graphify-serve
rm ~/.config/systemd/user/graphify-serve.service
systemctl --user daemon-reload
```
Port 8083 closes. Obsidian vault still works locally from the cloned repo — just no auto-sync.

---

### Phase 5 — Consolidation (optional, after 2–4 weeks)

Only do this after the graph has proved its value in practice.

1. **Trim ARCHITECTURE.md** — remove sections now covered better by the graph (code-level structure). Keep mission, principles, high-level component overview. PR to `idea` repo.
2. **Document refresh workflow** in `PROCESS.md` — when to re-read GRAPH_REPORT.md, how to trigger a manual graph rebuild.
3. **Evaluate** whether `graphify-out/cache/` is worth gitignoring on the runner (saves ~5–10 MB per commit but adds one rebuild pass on changed-file detection).
4. **Review graph quality** — look at GRAPH_REPORT.md after 3–5 merges: are the god nodes sensible? Are the suggested questions relevant? If not, tune `.graphifyignore` to exclude noise.

---

### Full Pi change summary

| Phase | Item | Location | Type | Rollback |
|-------|------|----------|------|----------|
| 1 | None | — | — | — |
| 2 | AGENTS.md additions | Each agent repo | +text block per agent | Remove lines via PR |
| 3 | Sparse clone | `/home/pi/graphify-cache/` | New directory | `rm -rf /home/pi/graphify-cache` |
| 3 | Crontab entry | `pi` user crontab | +1 line | `crontab -e`, delete line |
| 3 | Pull log | `/home/pi/idea/logs/graphify-pull.log` | New log file | `rm` the file |
| 4 | systemd unit | `~/.config/systemd/user/graphify-serve.service` | New file | `systemctl disable --now` + `rm` |
| 4 | TCP port 8083 | Network | Open port | Closed automatically when service is stopped |

**`/home/pi/idea` working directory: never touched by any of the above.**

**No changes to:**
- `~/.openclaw/openclaw.json`
- Any existing systemd services (OpenClaw, MC Docker, etc.)
- Any existing crontab entries
- `/home/pi/idea/platform/` (MC Docker stack)
- Python packages on the Pi (graphify builds and runs in GitHub Actions, not on the Pi)
- Tailscale config
- Git repo structure of any existing repos

### Fallback to current working state (complete removal)

If graphify is removed entirely after all 4 phases:

```bash
# Pi: remove crontab line
crontab -e  # delete the graphify line

# Pi: remove HTTP service
systemctl --user disable --now graphify-serve
rm ~/.config/systemd/user/graphify-serve.service
systemctl --user daemon-reload

# Pi: remove sparse clone and pull log
rm -rf /home/pi/graphify-cache
rm /home/pi/idea/logs/graphify-pull.log  # optional

# GitHub: remove Actions workflow
# → delete .github/workflows/graphify.yml via PR

# GitHub: remove graphify-out/ from idea repo
# → delete graphify-out/ and .graphifyignore via PR

# Agent repos: remove AGENTS.md additions
# → one-line PR to each agent repo
```

After removal: all agents revert to exactly current behaviour. No data is lost — CONTEXT.md and ARCHITECTURE.md were never removed (Phase 5 trim is optional and reversible via git history). Sessions resume as if graphify never existed.

---

## 9. Verdict

**Recommended to implement.** The auto-updating cross-codebase knowledge graph directly addresses IDEA's biggest coordination bottleneck: agents working across repo boundaries without reliable, current structural context. One graph from the idea root is simpler and more powerful than per-repo graphs. GRAPH_REPORT.md replaces (rather than supplements) manually-maintained architecture docs, keeping session-start token cost flat while improving quality and currency.

**One honest caveat:** OpenClaw's session-start-only mechanism is less powerful than the PreToolUse hook available to Claude Code users. We get structural orientation at session start; we don't get dynamic subgraph serving mid-session. That's still a meaningful improvement over the status quo, and the MCP server is available for deep sessions that need more.

The first build produces immediately visible output — `graph.html` alone is worth opening. API cost is one-time and modest. Subsequent rebuilds cost nothing for code.

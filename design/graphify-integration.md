# Design: Graphify Integration into IDEA Platform

**Status:** Proposal  
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

**The core mechanism — compressed subgraph serving:** when an agent asks a question, a PreToolUse hook fires before any file-read. The agent reads the graph map first, identifies relevant nodes, and receives a focused subgraph of ~300 tokens — instead of dumping 52 files into the context window. This is not static context injection; the graph acts as a queryable index.

**Graphify is a skill, not a standalone orchestrator.** The coding assistant (OpenClaw, Claude Code, Codex, etc.) is the runtime. Graphify dispatches extraction subagents in parallel, validates output against a schema, and merges it into the graph.

**Benchmarked token reduction:** 71.5x fewer tokens per query vs reading raw files (52-file mixed code+docs+images corpus). Token savings compound with corpus size.

Every edge carries a provenance tag — `EXTRACTED` (AST-derived, confidence 1.0), `INFERRED` (LLM-derived, with confidence score), or `AMBIGUOUS`. You always know what was found vs guessed.

---

## 2. Why This Is Relevant to IDEA

### 2.1 The cross-agent coordination problem

IDEA's five agents cannot talk to each other directly. Cross-agent coordination happens through shared files — CONTEXT.md, ARCHITECTURE.md, design docs. This means every agent reading a foreign codebase (Axle reading Console types, Pixel reading Engine events, Atlas reviewing a PR across both) must either:

- Read many files cold, burning tokens and time, or
- Rely on ARCHITECTURE.md being current (it lags)

A committed, auto-updating knowledge graph in the `idea/` repo gives every agent an instant structural map of the whole platform — without any agent needing to maintain it manually. Instead of cold-reading files, the PreToolUse hook serves the relevant 300-token subgraph on demand.

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

### 3.2 Hook only — no session-start index injection

**Question:** Is there still a need to load the index into each agent once it has access to the graph? Weight against token economy.

**Answer: No — the PreToolUse hook makes session-start injection redundant and wasteful.**

Here is what each mechanism does:

**`graphify claw install` (session-start injection):**
Writes a config that tells OpenClaw to inject `GRAPH_REPORT.md` into agent context at the start of every session. That's ~500–1,000 tokens of overhead per session, regardless of whether the agent will do any graph-related work. On the Pi, this adds up.

**PreToolUse hook:**
Fires before every file-read call. The agent reads the graph map, identifies relevant nodes, and receives only the focused 300-token subgraph for that specific query. Zero overhead on sessions that don't touch the graph. Exact context for sessions that do.

**Verdict:** The hook is strictly better for token economy. Inject nothing at session start. Agents query the graph on demand via the hook. `GRAPH_REPORT.md` remains a valuable human document (for Koen to review the graph structure) but should not be auto-injected into every agent session.

**Exception:** If an agent is starting a large cross-codebase refactor and explicitly wants orientation, it can read `GRAPH_REPORT.md` deliberately. That's the right time to read it — not as boilerplate on every session start.

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

```bash
graphify hook install   # post-commit git hook in idea/ repo
```

Code changes → instant AST rebuild, no API cost.  
Doc/image changes → hook notifies to run `graphify . --update`, which only reprocesses changed files.

The graph in `graphify-out/` is committed. Every agent pull gets the latest map.

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

**Graph can go stale without the hook.** The post-commit hook mitigates this. Agents should treat the graph as advisory and verify against source for critical decisions.

**Build should run on a dev machine or CI, not the Pi.** The Python extraction CLI is compute-light but shouldn't run on the Pi during active sessions. Best practice: run `/graphify .` on a dev machine after significant changes, commit the output, let the Pi pull it. The git hook can also run on the dev machine.

**`graph.json` size.** Can grow on large codebases. Recommend committing `graph.json`, `GRAPH_REPORT.md`, and `obsidian/`. Gitignore `manifest.json`, `cost.json`, and optionally `cache/` (large, safe to regenerate).

---

## 8. Implementation Plan

### Phase 1 — Build and validate (one session, Koen or Atlas)
1. On a dev machine with Python 3.10+: `pip install graphifyy`
2. Clone `koenswings/idea` with all submodules
3. Create `.graphifyignore` at root (see §3.1)
4. Run `/graphify .` from idea root — inspect `GRAPH_REPORT.md` and `graph.html`
5. If useful: commit `graphify-out/` to a branch, open PR for review

### Phase 2 — Agent integration (Atlas)
1. `graphify claw install` on the Pi — installs the PreToolUse hook
2. Do **not** configure session-start injection (see §3.2)
3. Test: Axle session working on Engine asks about Console boundary → verify hook serves subgraph

### Phase 3 — Mac access (Atlas)
1. Add `graphify-serve.service` systemd unit on Pi (port 8083, serves `graphify-out/`)
2. Enable and start: `systemctl --user enable --now graphify-serve`
3. Koen pulls idea repo on Mac → opens `graphify-out/obsidian/` as Obsidian vault
4. Verify both access points work over Tailscale

### Phase 4 — Automation
1. `graphify hook install` in idea repo → auto-rebuild on commit
2. Add `graphify-out/manifest.json` and `graphify-out/cost.json` to `.gitignore`
3. Document refresh workflow in `PROCESS.md`

---

## 9. Verdict

**Recommended to implement.** The token savings and automatic cross-codebase navigation directly address IDEA's biggest coordination bottleneck. Native OpenClaw support keeps integration friction low. The hook-only approach preserves token economy. One graph from the idea root is simpler and more powerful than per-repo graphs.

The first build produces immediately visible output — `graph.html` alone is worth opening. API cost is one-time and modest. Subsequent rebuilds cost nothing for code.

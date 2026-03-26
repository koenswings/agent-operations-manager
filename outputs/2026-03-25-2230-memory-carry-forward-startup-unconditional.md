> **Task/Question:** Go. And also make the startup read unconditional: agents have to read when they start and not wait for an /init [re: memory carry-forward mechanism gap analysis]

## Changes made

### 1. Memory write timing — all 5 agents + design doc

**Old instruction:** "Write important context, decisions, and lessons to `memory/YYYY-MM-DD.md` each session."

**New instruction:** "After each substantive exchange, append key points to `memory/YYYY-MM-DD.md`. Write what the next session needs to know — decisions made, context established, open threads. Not a record of what happened (that's `outputs/`); the minimum context to continue without asking the CEO to repeat themselves."

Key changes:
- "each session" → "after each substantive exchange" (timing is now explicit)
- Added purpose distinction: memory ≠ outputs; memory is forward-looking

### 2. Unconditional startup reads — all 5 agents + design doc

**Old instruction:** "Before doing anything else: [list]"

**New instruction:** "Read these at session start — before your first response, without exception. Do not wait for /init."

The `/init` section in the design doc updated to be explicit: startup reads are unconditional; `/init` is recovery-only for cases where the normal startup didn't happen.

### 3. Stale CLAUDE.md reference removed from Axle (engine-dev)

Axle's startup checklist still listed "Read `CLAUDE.md` — project conventions, architecture, key commands." That content was in the old project-guide CLAUDE.md which no longer exists. Removed from:
- `agent-engine-dev/AGENTS.md`
- Design doc per-agent startup checklists table

### Files changed
- `design/virtual-company-design.md` — Agent Memory section + startup table + /init description
- `agents/agent-engine-dev/AGENTS.md`
- `agents/agent-console-dev/AGENTS.md`
- `agents/agent-site-dev/AGENTS.md`
- `agents/agent-programme-manager/AGENTS.md`
- `agents/agent-operations-manager/AGENTS.md`

### PRs
- All 4 operational agents: updated on existing memory/updates PRs (#13, #9, #9, #16)
- Atlas: memory/updates branch (open PR #4)
- idea: updated on PR #5 (`docs/cross-agent-requests`)

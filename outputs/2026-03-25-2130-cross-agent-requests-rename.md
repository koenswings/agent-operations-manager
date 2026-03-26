> **Task/Question:** OK go ahead. Implement and document in the company design doc [re: renaming auto-review tasks to cross-agent requests, expanding to cover opinions/questions/feasibility]

## What changed

**Concept renamed:** "auto-review tasks" → "cross-agent requests"

The old name implied the mechanism was only for quality review. The new name reflects that any agent can ask any other agent for any type of input: a review, a domain question, an opinion, or a feasibility check.

## Design decisions

**Tag:** `auto-review` → `cross-agent` — consistent with new name, used in script and MC task creation.

**Request types** (typed prefix in task title):
- `Review:` — quality check on a PR, design doc, or proposal
- `Question:` — domain question (e.g. "Does the Engine expose usage metrics?")
- `Opinion:` — input on a decision touching another agent's domain
- `Feasibility:` — technical or field viability check

**Cycle prevention** unchanged in mechanism; guard #1 updated from "reviewer SOUL.md" to "target agent's AGENTS.md" since not all requests are to reviewers.

**Depth-1 constraint preserved** — regardless of request type, no chaining. A question gets one answer; an opinion gets one response. No follow-up tasks.

## Files changed

- `design/virtual-company-design.md` — work cycle diagram, mechanism section (full rewrite), cron table, Multi-Agent Dialogue section
- `scripts/check-new-tasks.sh` — tag filter changed, Veri removed, Atlas commented pending board ID, agent IDs updated to lead-* format, prompt updated

## Flags raised

**Atlas board:** Atlas has no MC board, so cross-agent requests to Atlas cannot currently be auto-triggered. Veri's board (d0cfa49e) could be repurposed, or a new board created. The script has Atlas commented out until board ID is confirmed. Koen needs to verify via MC UI.

**Script agent IDs:** The old script used UUID-style agent IDs that don't match the current lead-* format in openclaw.json. Updated to lead-* format; needs testing in production.

## PR

idea PR #5 (`docs/cross-agent-requests`)

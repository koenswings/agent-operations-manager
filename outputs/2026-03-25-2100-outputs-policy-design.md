> **Task/Question:** I would like to produce output files for everybody. Not to be read on every start but for auditing purposes. Something like: Every substantive response MUST be written to a file. This is not optional. - Write to outputs/YYYY-MM-DD-HHMM-<topic>.md - Start every output file with > **Question:** <the user's exact question> - Then commit and push the output file immediately to a separate output file branch on which one pr is continuously updated and approved once and a while by the ceo - No exceptions for "short" responses — truncation is unpredictable - This applies to EVERY session, EVERY response, EVERY interface / Yes and implement it.

## Design discussion

Koen proposed an output file system for all agents — every substantive response written to `outputs/`, committed and pushed immediately. This reinstates a pattern that existed for Compass and was lost in the restructure.

## Decisions made

**Threshold:** substantive = any response with analysis, a decision, a plan, a recommendation, or a work product. Exempt: one-liner confirmations, status ACKs, pure yes/no answers. Rationale: "every response" creates noise and overhead on a Pi; a clear threshold lets all agents apply the rule consistently.

**Header:** `> **Task/Question:**` — covers both commands and questions (Koen doesn't always ask questions).

**Branch:** `memory/updates` — not a separate outputs branch. Combining outputs and memory on one branch keeps PR overhead low. One PR per agent, merged on CEO's schedule.

**Filename slug:** 2–4 words, kebab-case, describing content not format.

**Commit message format:** `outputs: YYYY-MM-DD <topic>`

## Implementation

- `design/virtual-company-design.md` — Session Documentation section rewritten with precise policy
- All five agents' `AGENTS.md` — `## Outputs` section added
- This file — first output under the new policy
- PRs: idea `docs/outputs-policy` → PR to open; all five agents' `memory/updates` → PRs to open

## What was NOT done

- No separate outputs branch (merged with memory/updates per decision above)
- No change to the "every response" direction — threshold defined instead to make it practical

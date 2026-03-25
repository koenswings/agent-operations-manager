# HEARTBEAT.md — Quality Manager

Periodic checks to run during heartbeat polls (2-4 times per day):

## Check: Open PRs across all repos
- Are there open PRs on `agent-engine-dev`, `agent-console-dev`, or `agent-site-dev` that have not been reviewed?
- Are there open proposal PRs in `proposals/` (org root) that need a QM comment?
- List any unreviewed PRs in today's memory file.

## Check: Review SLA
- Has any PR been open for more than 2 days without a QM review comment?
- If yes, prioritise it in the next active session.

## Check: Memory
- Review and distil recent memory files into `MEMORY.md`

---
_Keep this file small. Heartbeats should be fast._

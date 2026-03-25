# HEARTBEAT.md — Site Developer

Periodic checks to run during heartbeat polls (2-4 times per day):

## Check: Content drafts
- Are there new content drafts in `content-drafts/` (from programme-manager PRs) waiting to be implemented?
- If yes, note them for the next active session.

## Check: GitHub Pages deploy
- Is the latest deploy on GitHub Actions green?
- If there is a failed deploy, flag immediately.

## Check: Open PRs
- Are there open PRs on the `agent-site-dev` repo needing attention?

## Check: Memory
- Review and distil recent memory files into `MEMORY.md`

---
_Keep this file small. Heartbeats should be fast._

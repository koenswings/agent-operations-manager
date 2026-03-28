> **Task/Question:** Please add this summary to the design doc. For now, I would like to stay away from using MC - so you can describe the Mission Control section as an alternative for the current flow which is using telegram and github. You can add this summary in that section. Am I right (please confirm) the only feature we currently do use are the boards themselves with exports to an .md for traceability in Github. This would allow me to explore using MC in the future so that we can switch to MC with a good understanding of its usability when we think it would be more useful.

## Confirmation

Yes, correct. The only MC features currently in active use are:
1. Kanban boards as task tracker (create tasks, update status, view across agents)
2. BACKLOG.md export — `scripts/export-backlog.sh` queries MC and writes a git-traceable snapshot

Everything else (agent chat, plan approval, activity timeline, escalation, messaging) happens in Telegram and GitHub. MC is the record, not the channel.

## Design Doc Changes (idea PR #9, chore/post-migration-updates)

### Mission Control section
- Added "What We Actually Use" subsection up front — two-item list, plain language
- Explicit statement: "MC is the record, not the channel"
- "How task management works" rewritten — removed false claim that plan approval happens in MC; corrected to state agents read BACKLOG.md and all interaction happens in Telegram
- Feature Map renamed to "Feature Map — Available for Future Use" with clear framing that none are currently active
- Full feature map preserved for future reference: Approvals, Board Memory, Board Group Memory, Activity Feed, Agent Roster & Lifecycle, Nudge, Onboarding, Webhooks, Gateways API, Escalation & Messaging, Skills Marketplace, Souls Directory, Dashboard Metrics

### CEO Tools & Daily Workflow section
- Tool stack rewritten as plain list (not a table) with correct role for each tool
- MC correctly described as "task register" not "approval management" or "primary interface"
- "Using Mission Control + Agent Tabs" replaced with "How a Work Cycle Works" — 5-step Telegram-first flow
- "Which agent to message" replaces "Which agent tab to use" — reflects Telegram as the surface
- "A Typical CEO Day" replaces "A Typical CEO Interaction" — removes MC chat references throughout

## My Assessment

This is the right approach. The Telegram + GitHub workflow is simple and works from a phone. Adding MC interaction layers before the org is properly running would add complexity without clear payoff. The Feature Map gives us a solid reference for when the question "should we use more of MC?" comes up — we'll know exactly what we're choosing between and why.

The most likely features to adopt first, when the time is right:
- Webhooks (closes the GitHub↔MC loop automatically)
- Board Memory (durable operational state visible in MC UI)
- Lead Broadcast (Atlas → all agents in one message)

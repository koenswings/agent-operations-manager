> **Task/Question:** Right now, my approval workflow is as follows (please document this in the design doc). The agent makes a PR - I read it and give the agent any comment about it in a Terminal chat - the agent adapts the PR until it is right - I then finally approve. Can this MC approval feature be setup to do something similar? And what do you mean by "it adds friction"

## Assessment

### Your Current PR Review Loop (now documented)

1. Agent completes work on feature branch → opens PR
2. You read the PR → send feedback to agent in chat (Telegram or MC)
3. Agent pushes update to same branch → PR refreshes automatically
4. Repeat steps 2–3 until output is right
5. You merge on GitHub

This is correct and already the right pattern. The design doc previously described it but didn't make the iteration loop explicit. Fixed.

### MC Approvals vs PRs

| | PR review loop | MC Approvals |
|---|---|---|
| **What it gates** | Code/doc change before it lands in `main` | Runtime action agent is about to take |
| **Review surface** | The diff (code, text, structure) | Agent's stated intent + JSON payload |
| **Iteration** | Natural — agent pushes, PR updates | None built in — single request |
| **Audit trail** | Permanent in git history | MC database only |
| **Trigger** | Agent opens PR | Agent makes API call pre-action |

**MC Approvals cannot replicate the PR review workflow.** They solve a different problem: gating irreversible runtime actions (send external email, delete data, deploy). They're useful when you want a structured audit trail for decisions that don't go through git.

### On "adds friction"

Imprecise language. What I meant: MC approvals require extra ceremony (agent posts API request, you act in MC UI, agent polls for result) vs the conversational flow we have today. Not inherently bad — for truly irreversible actions (Marco sending a briefing to an external contact, Axle wiping a disk partition), a formal gate with an audit trail is good discipline. Just not the right tool for PR iteration.

## Design Doc Changes

- `## CEO Approval — Two Layers` → **Three Layers**; Layer 2 now documents the full PR iteration loop; Layer 3 added for MC Approvals (runtime action gating)
- Step table: Review row updated to reflect Telegram/MC feedback + "repeat until right" iteration
- Redundant "PR review happens in chat" note removed (folded into Layer 2 description)

Committed to `chore/post-migration-updates` (idea PR #9, now 7 commits).

> **Task/Question:** OK. Go ahead and give me at the end a summary of the existing tasks on the boards that you will clean up.

## Changes Made

### idea repo — PR #20 ([fix/koen-hub-cross-agent](https://github.com/koenswings/idea/pull/20))

**design/virtual-company-design.md:**
- Work cycle diagram: replaced MC-task + cron flow with Telegram relay pattern
- Key mental model: updated ("all cross-agent communication relayed through you")
- Cross-agent section: replaced MC board task protocol with "Cross-agent communication" (Koen-as-hub)
- MC board mechanism preserved as documented future phase ("Cross-agent communication via MC boards")
- "How agents coordinate": replaced "MC tasks" as primary with "Telegram relay" as primary
- Automated tasks table: check-new-tasks.sh marked "currently disabled"

**scripts/check-new-tasks.sh:**
- Added `exit 0` at top with a clear DISABLED comment and re-enable instructions

### Agent AGENTS.md updates (on each agent's memory/updates branch)

All 5 agents' cross-agent sections replaced with the Koen-as-hub protocol:

> Send Koen a message in your own Telegram group:
> 📨 **For [AgentName]:** [self-contained message]
> Koen forwards manually. Target agent responds; Koen relays back.

| Agent | Repo | Branch | PR |
|-------|------|--------|----|
| Axle | agent-engine-dev | memory/updates | [#18](https://github.com/koenswings/agent-engine-dev/pull/18) |
| Pixel | agent-console-dev | memory/updates | [#11](https://github.com/koenswings/agent-console-dev/pull/11) |
| Beacon | agent-site-dev | memory/updates | [#11](https://github.com/koenswings/agent-site-dev/pull/11) |
| Marco | agent-programme-manager | memory/updates | [#19](https://github.com/koenswings/agent-programme-manager/pull/19) |
| Atlas | agent-operations-manager | memory/updates | [#9](https://github.com/koenswings/agent-operations-manager/pull/9) |

Atlas note: PR reviews still go as GitHub PR comments; only non-PR cross-agent communication uses Telegram relay.

## MC Tasks Cleaned Up

5 tasks deleted across 3 boards:

**Axle's board** — 3 tasks (Pixel's unanswered questions):
1. `[From Pixel] Question: Engine software update mechanism — how does it work?` (6b9c9e71)
2. `[From Pixel] Question: Serving Console production web app from Engine` (9b8c6478)
3. `[From Pixel] Question: GET /api/store-url — Engine endpoint` (7b45ffbf)

**Pixel's board** — 1 task (Atlas fix notification, no longer needed):
4. `[From Atlas] Info: Cross-board write fix applied — ready to use` (7151bace)

**Atlas's board** — 1 task (old cross-agent review request):
5. `Test setup design review — automated tests, simulate disk dock/undock` (318dfd03)

## Note for Koen

- Pixel still has those 3 questions for Axle unanswered — she can now relay them via you using the new format
- check-new-tasks.sh is disabled in code; takes effect after PR #20 is merged and `git pull` is run on the Pi
- All AGENTS.md changes take effect after the memory/updates PRs are merged

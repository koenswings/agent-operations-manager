# IDEA Platform: Migration to Grok Bot

**Author:** Atlas  
**Date:** 2026-09-19  
**Status:** Migration plan — superseded by `docs/grok-bot-setup.md` once migration is complete

This document records the migration steps from the OpenClaw setup to the Grok Bot setup described in `docs/grok-bot-setup.md`.

---

## Rollback

Before any migration changes, tag the current state of `koenswings/idea`:

```bash
cd /home/pi/idea
git tag v-openclaw-final
git push origin v-openclaw-final
```

To restore the prior setup at any time: `git checkout v-openclaw-final`, restart OpenClaw.

---

## Phase 0 — Prepare (Atlas)

**koenswings/idea cleanup PR:**
- Rewrite `CONTEXT.md` for Grok Bot (condensed mission + team structure)
- Delete `ROLES.md` — content absorbed into CONTEXT.md
- Delete `platform/`, `skills/`, `standups/`, `graphify-out/`, stale backup scripts, `prompting-guide-opus.md`
- Create `fleet-state.json` (all Pis idle)
- Add `docs/grok-bot-setup.md` to `docs/INDEX.md`

**Dev repo cleanup:**
- Commit all outstanding identity/memory files across all agent repos
- Final MC pg_dump archived to agent-identities repo
- `agent-engine-dev`: create `design/`, move `docs/SOLUTION_DESCRIPTION.md` → `design/SOLUTION_DESCRIPTION.md`, update `docs/INDEX.md`
- `agent-console-dev`: create `design/` folder
- `agent-app-dev`: `design/` already exists

**MilkWise extraction:**
- Move MilkWise design docs from `agent-app-dev/design/milkwise/` → `baby-milk-tracker/design/`
- Move `apps/app-milkwise/` from `agent-app-dev` to new repo `koenswings/app-milkwise`
- Update App Dev Bot description to list `koenswings/app-milkwise` in its maintained repos

**deploy-fleet.sh update:**
- Remove wizardly-hugle as a deploy target (it is being retired)

OpenClaw continues running throughout Phase 0.

---

## Phase 1 — SuperGrok Link ✅ Done

SuperGrok Heavy linked to Cursor. Grok Bot active at Heavy+ tier.

---

## Phase 2 — Grok Build + Runners on Pis (Atlas, ~1 hour)

On each Pi (idea01–idea04):

```bash
curl -fsSL https://x.ai/cli/install.sh | bash
grok auth login   # Koen authenticates via browser once
```

GitHub Actions self-hosted runner: Atlas generates registration tokens via GitHub API. Runners run as systemd services, labelled `idea01`–`idea04`.

Create initial `fleet-state.json` in `koenswings/idea` (all Pis idle).

Test: trigger a workflow targeting `idea01`, verify Grok Build runs.

---

## Phase 3 — Grok Bot Setup (Koen, ~1 hour)

1. Create 5 Bots in Grok Bot: Lead, Engine Dev, Console Dev, App Dev, Ops
2. Paste Bot descriptions from `docs/grok-bot-setup.md` Section 7 into each Bot
3. Install GitHub connector → connect koenswings account
4. Create **IDEA Design Review** group chat (all 5 Bots)
5. Create **IDEA Programme** group chat (Lead + Marco + App Dev)
6. Test:
   - *Bug path:* tell Lead Bot a bug → issue created → Dev Bot triggers runner → QC → Ops deploys → URL
   - *Feature path:* tell Lead Bot a feature → Design Review group → proposal PR → koenswings/idea

---

## Phase 4 — Parallel Run (~1 week)

OpenClaw stays on as fallback. Koen uses Grok Bot for new work. Lead Bot documents any gaps as GitHub issues.

---

## Phase 5 — wizardly-hugle Retirement (when Koen says go)

- [ ] All GitHub repos committed and clean
- [ ] Final MC pg_dump archived
- [ ] ANTHROPIC_API_KEY noted → add to Grok Build config on each Pi
- [ ] XAI_API_KEY → add to Grok Build config on each Pi
- [ ] Tailscale node removed from tailnet admin console
- [ ] `systemctl --user stop openclaw`
- [ ] `docker compose -f /home/pi/idea/platform/compose.yaml down`
- [ ] Machine powered off or repurposed

---

## What Goes Away vs What Stays

| Item | Status |
|------|--------|
| wizardly-hugle | Gone |
| OpenClaw | Gone |
| Mission Control | Gone |
| Telegram groups | Gone |
| Graphify | Gone |
| ROLES.md, BACKLOG.md, standups/ | Gone (content in CONTEXT.md or archived) |
| platform/, skills/ in idea repo | Gone |
| Daily memory files, /flush | Gone — Grok Bot handles memory |
| Nightly backup cron | Gone — GitHub is source of truth |
| | |
| SuperGrok Heavy | ✓ Keep |
| Pi fleet (idea01–04) | ✓ Keep |
| GitHub repos | ✓ Keep |
| Tailscale | ✓ Keep |
| AGENTS.md files | ✓ Keep — Grok Build operational manuals |
| koenswings/idea (cleaned up) | ✓ Keep |

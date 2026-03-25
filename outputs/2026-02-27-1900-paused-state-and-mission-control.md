> **Question:** Can we proceed with OpenClaw in a paused state? Is there a Mission Control installation step?

---

## OpenClaw paused state — yes, fully compatible

All remaining steps up to the first live CEO session can be done with OpenClaw down:

| Step | Needs OpenClaw running? |
|------|------------------------|
| 11 — Copy sandbox files | No — file copy only |
| 12 — Restart / verify | Briefly up, then back down |
| **NEW** — Deploy Mission Control | Needs `docker compose up` to test, then back down |
| 13 — Branch protection | No — GitHub only |
| 14 — Pair browser with agents | Yes — CEO present required |
| 15 — BOOTSTRAP sessions | Yes — CEO present required |

**Recommended approach:**
- Bring OpenClaw down now
- Complete steps 11 and 13 offline
- For step 12 + Mission Control: bring up once, verify both are healthy, bring back down
- Steps 14 and 15: CEO does these live when ready

---

## Mission Control — gap in the numbered steps

You're right. Mission Control deployment is in the proposal backlog but was not in the
numbered "What Needs to Happen" sequence. It should be there.

**Current state:** Mission Control is NOT running. Only `openclaw-gateway`, `tailscale`,
and `ssh-proxy` are up. The proposal's Phase 1 note that "Mission Control already running"
was a decision (to use it from day one), not a deployment fact.

**Where it fits:** Mission Control must be added to `compose.yaml` and started before
agents are bootstrapped — the CEO needs it running to use it as the primary interface
for plan approval and task dispatch (steps 14+).

### Proposed updated sequence

| # | Step | Status |
|---|------|--------|
| 10 | ✅ Apply `openclaw.json` | Done |
| 11 | Copy sandbox files to each agent's OpenClaw sandbox | Next |
| 12 | Bring OpenClaw up briefly — verify all 6 agents visible — bring back down | — |
| **12a** | **Deploy Mission Control** — add MC service to `compose.yaml`; bring up; configure board hierarchy; bring back down | **NEW** |
| 13 | Branch protection on `main` across all repos | — |
| 14 | CEO live: pair browser with each agent in OpenClaw UI | CEO session |
| 15 | CEO live: BOOTSTRAP sessions for all new agents | CEO session |

### What Mission Control deployment involves

1. Add MC service block to `/home/pi/openclaw/compose.yaml` (the `abhi1693/openclaw-mission-control` image)
2. Set `LOCAL_AUTH_TOKEN` (min 50 chars) in a new secrets file
3. `docker compose up -d` → verify MC accessible at port 4000
4. In the MC UI: create board hierarchy — IDEA org → Engineering (engine-dev, console-dev, site-dev, quality-manager) + HQ (programme-manager) boards
5. Bring back down until CEO is ready for the live session

---

## Action taken

- OpenClaw brought down: `cd /home/pi/openclaw && sudo docker compose down`
  (tailscale and ssh-proxy stay up — only `openclaw-gateway` stops)
- Step 12a added to the numbered sequence
- Build log updated

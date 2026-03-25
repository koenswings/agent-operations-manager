> **Question:** Telegram post-fix status — agent responded, MC was down

## Outcome

Telegram is fully operational end-to-end after adding the `bindings` array to `openclaw.json`.

**First agent response received via Telegram:**
> *Axle here — alive and monitoring. ⚙️ API still down at 16:58 UTC — 172.18.0.1:8000 has been unreachable since 04:11 UTC today (~12h 47m and counting)...*

This confirmed two things simultaneously:
1. The Telegram binding fix worked — DMs from the CEO are reaching agents
2. Mission Control had been down for ~13 hours (since ~04:11 UTC 2026-03-19)

## MC Outage

- **Cause:** All MC containers had stopped (backend, frontend, postgres, redis, webhook-worker). Only `openclaw-gateway` was running.
- **Fix:** `cd /home/pi/openclaw/mission-control && sudo docker compose up -d`
- **Restored:** backend (`:8000`) and frontend (`:4000`) both returned 200 within ~5s
- **Agent recovery:** Axle self-recovered — "Pre-flight passed. Board is live — 12 tasks in inbox."

## Recommended follow-up

The MC stack has no restart policy configured. If the Pi reboots or containers crash, MC will stay down indefinitely. Add `restart: unless-stopped` to all services in `/home/pi/openclaw/mission-control/compose.yml`, or configure a systemd service that starts `docker compose up -d` on boot.

## Telegram config reference

Bot: `@Idea911Bot` (ID: `8496871029`)  
CEO allowlist: `["8320646468"]`  
Config: `/var/lib/docker/volumes/openclaw_openclaw-data/_data/openclaw.json`  
Bindings added: 5 entries under top-level `bindings[]`, one per board lead agent, `match: { channel: "telegram" }`

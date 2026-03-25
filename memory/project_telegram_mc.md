---
name: Telegram channel and MC restart policy
description: Telegram is live via @Idea911Bot; MC stack has no restart policy and needs one
type: project
---

Telegram is fully operational via OpenClaw's native channel system — no custom bridge needed.

**Why:** OpenClaw's gateway includes a Telegram plugin. The CEO already had the bot token and allowlist configured in `openclaw.json`. The only missing piece was the `bindings` array routing DMs to agents.

**Fix applied 2026-03-19:** Added top-level `bindings[]` to `/var/lib/docker/volumes/openclaw_openclaw-data/_data/openclaw.json` — 5 entries, one per board lead agent, each with `match: { channel: "telegram" }`.

**How to apply:** Discovered on 2026-03-19 — all 5 board leads bound to `telegram` channel (broadcast mode when equal specificity; narrow with peer/group filters if needed).

**MC outage discovered same session:** MC had been down ~13h (since 04:11 UTC). All containers had stopped; only openclaw-gateway was running. Fixed with `docker compose up -d` in `/home/pi/openclaw/mission-control/`.

**Pending:** MC compose.yml has no `restart: unless-stopped` policy — containers will not auto-recover after Pi reboot or crash. This is a reliability gap that should be fixed.

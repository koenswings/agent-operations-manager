# OpenClaw Recovery Briefing
_Written by Atlas — keep this up to date after any config change or restart_

## Situation
You are reading this because OpenClaw failed to restart after a config change, OR because a sub-agent is experiencing model override issues.
This doc tells you everything you need to get it back up.

---

## What was being changed (last incident: 2026-09-09/15)
The `agents.defaults.models` section in `~/.openclaw/openclaw.json`.
Specifically: the opus model entry alias/ID.

Previous broken ID: `claude-opus-4-8` (never existed)
Correct canonical ID: `claude-opus-4-5-20251101`

---

## The config file
Path: `/home/pi/.openclaw/openclaw.json`

### Critical rules for this file:
- `gateway.bind` MUST be `"loopback"` — **never "lan" or "all"**
  - Tailscale Serve owns port 18789 on the LAN interface
  - Setting bind to "lan" causes an instant crash-loop (NRestarts climbs, service dies)
- `session.reset.mode` must be `"idle"` (not daily)
- Do NOT invent model IDs — verify against Anthropic API first
- Canonical model IDs (verified 2026-09-15): `claude-opus-4-5-20251101`, `claude-sonnet-4-6`, `claude-haiku-4-5`, `claude-fable-5`

### To restore the last known-good config:
```bash
cp ~/.openclaw/openclaw.json.bak.20260629 ~/.openclaw/openclaw.json
systemctl --user restart openclaw
systemctl --user status openclaw
```

If that backup is stale, here is a safe minimal models section to paste in:
```json
"models": {
  "anthropic/claude-sonnet-4-6": {
    "alias": "sonnet",
    "params": { "cacheRetention": "short" }
  },
  "anthropic/claude-haiku-4-5": {
    "alias": "haiku",
    "params": { "cacheRetention": "short" }
  },
  "anthropic/claude-fable-5": {
    "alias": "fable",
    "params": { "cacheRetention": "short" }
  },
  "anthropic/claude-opus-4-5-20251101": {
    "alias": "opus",
    "params": { "cacheRetention": "short" }
  }
}
```

---

## OpenClaw service management
```bash
# Check status
systemctl --user status openclaw

# Restart
systemctl --user restart openclaw

# View live logs
journalctl --user -u openclaw -f

# Check for port conflicts (the main crash cause)
ss -tlnp | grep 18789
```

---

## If the service is crash-looping
```bash
# See what's happening
journalctl --user -u openclaw -n 50

# Common cause: bind conflict with Tailscale
# Fix: ensure openclaw.json has gateway.bind = "loopback"
grep '"bind"' ~/.openclaw/openclaw.json

# If bind is wrong, fix it:
sed -i 's/"bind": "lan"/"bind": "loopback"/' ~/.openclaw/openclaw.json
systemctl --user restart openclaw
```

---

## If an agent gets "Model override not allowed for this agent"

This is a session-store stale model override issue. It happens when:
1. A model was stored in the session store with an ID (e.g. `claude-opus-4-8`)
2. That ID was later changed/corrected in `openclaw.json` (e.g. to `claude-opus-4-5-20251101`)
3. The stored override no longer matches the allowlist → OpenClaw resets it with that error message

**Default model:** All agents should be on `claude-sonnet-4-6` by default. Only set opus as a session override when Koen explicitly requests it (e.g. `@opus` inline). Do NOT set opus as a persistent override across all agents.

**Fix (no restart needed):**
```bash
# Edit the session store directly
python3 -c "
import json

path = '/home/pi/.openclaw/agents/<agent-id>/sessions/sessions.json'
with open(path) as f:
    data = json.load(f)

for key, val in data.items():
    if isinstance(val, dict):
        for field in ['model', 'modelOverride']:
            old = val.get(field)
            # replace stale IDs
            if old in ['claude-opus-4-8', 'opus', 'claude-opus-4-6']:
                val[field] = 'claude-opus-4-5-20251101'
                print(f'Fixed {key}.{field}: {old} -> claude-opus-4-5-20251101')

with open(path, 'w') as f:
    json.dump(data, f, indent=2)
print('Done')
"
```
Replace `<agent-id>` with the agent id (e.g. `app-dev`, `operations-manager`, etc.)

**Root cause:** Sub-agents spawned with `model: "opus"` store the raw alias string in the session store.
When the alias resolves to a different canonical ID than what's in the allowlist, the check fails.

**Prevention:** Always use the full canonical model ID in `sessions_spawn`, not aliases:
- ✅ `anthropic/claude-opus-4-5-20251101`
- ❌ `opus` (alias — not resolved at spawn time)

---

## Agents and their Telegram groups
| Agent | ID | Telegram Group ID |
|-------|-----|------------------|
| Atlas (operations) | `operations-manager` | -5105695997 |
| Axle (engine dev) | `lead-6bddb9d2-c06f-444d-8b18-b517aeaa6aa8` | -5146184666 |
| Pixel (console dev) | `lead-ac508766-e9e3-48a0-b6a5-54c6ffcdc1a3` | -5187034968 |
| Beacon (site dev) | `lead-7cc2a1cf-fa22-485f-b842-bb22cb758257` | -5139661372 |
| Marco (programme) | `lead-3f1be9c8-87e7-4a5d-9d3b-99756c35e3a9` | -5141459717 |
| Kit (app dev) | `app-dev` | -5296497974 |

Agent session stores: `/home/pi/.openclaw/agents/<id>/sessions/sessions.json`

---

## MC (Mission Control)
- Docker: `docker compose -f /home/pi/idea/platform/compose.yaml ps`
- MC URL: `https://idea.tail2d60.ts.net:4000`
- MC API: `http://127.0.0.1:8000`

---

_If everything is broken and you need a fresh Claude session to debug:_
_Start at `journalctl --user -u openclaw -n 100` and look for the error line._
_The most common fix is restoring the backup config file + restarting the service._

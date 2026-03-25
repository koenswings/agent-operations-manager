> **Question:** I created the groups. What do I do to connect them to each agent?

## Connecting Telegram Groups to Agents

You created 5 Telegram groups — one per agent. To route each group to its agent, you need to:

1. **Get each group's chat ID** (if you don't have them already)
2. **Update `openclaw.json`** — two changes:
   - Add all group IDs to the `allowFrom` allowlist in the telegram channel config
   - Replace the generic `match: { channel: "telegram" }` bindings with group-specific ones

---

## Step 1 — Get the group chat IDs

Add the bot to each group. Then send a message in the group and call:

```
https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates
```

Look for `"chat": { "id": -xxxxxxxxxx }` — group IDs are negative numbers.

Your bot token: `8496871029:AAF1eO8...` (already in openclaw.json)

---

## Step 2 — Update openclaw.json

Once you have the 5 group IDs, edit `/var/lib/docker/volumes/openclaw_openclaw-data/_data/openclaw.json`:

### 2a — Add groups to the allowlist

```json
"telegram": {
  "dmPolicy": "allowlist",
  "botToken": "...",
  "allowFrom": [
    "8320646468",
    "-100XXXXXXXXX1",
    "-100XXXXXXXXX2",
    "-100XXXXXXXXX3",
    "-100XXXXXXXXX4",
    "-100XXXXXXXXX5"
  ],
  "groupPolicy": "allowlist",
  "streamMode": "partial"
}
```

### 2b — Replace the bindings with group-specific ones

Current bindings all match `{ "channel": "telegram" }` — that means all 5 agents respond to every message (broadcast). Replace with:

```json
"bindings": [
  {
    "agentId": "lead-6bddb9d2-c06f-444d-8b18-b517aeaa6aa8",
    "match": { "channel": "telegram", "peer": "-100XXXXXXXXX1" }
  },
  {
    "agentId": "lead-ac508766-e9e3-48a0-b6a5-54c6ffcdc1a3",
    "match": { "channel": "telegram", "peer": "-100XXXXXXXXX2" }
  },
  {
    "agentId": "lead-7cc2a1cf-fa22-485f-b842-bb22cb758257",
    "match": { "channel": "telegram", "peer": "-100XXXXXXXXX3" }
  },
  {
    "agentId": "lead-d0cfa49e-edcb-4a23-832b-c2ae2c99bf67",
    "match": { "channel": "telegram", "peer": "-100XXXXXXXXX4" }
  },
  {
    "agentId": "lead-3f1be9c8-87e7-4a5d-9d3b-99756c35e3a9",
    "match": { "channel": "telegram", "peer": "-100XXXXXXXXX5" }
  }
]
```

Map each group ID to its agent:
| Agent | Name | board ID prefix |
|-------|------|-----------------|
| Axle | Engine Dev | `lead-6bddb9d2` |
| Pixel | Console Dev | `lead-ac508766` |
| Beacon | Site Dev | `lead-7cc2a1cf` |
| Veri | Quality Mgr | `lead-d0cfa49e` |
| Marco | Programme Mgr | `lead-3f1be9c8` |

---

## Step 3 — Restart the OpenClaw gateway

```bash
cd /home/pi/openclaw/mission-control && docker compose restart openclaw-gateway
```

The new bindings take effect immediately after restart.

---

## What about DMs?

Your existing DM flow still works — DMs from `8320646468` (your Telegram user ID) hit all 5 agents in broadcast. You may want to keep one binding with no `peer` filter for your DMs (or remove them if you only want group-per-agent routing).

---

## Give me the group IDs and I'll generate the exact JSON for you.

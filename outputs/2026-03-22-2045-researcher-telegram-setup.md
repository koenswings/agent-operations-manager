> **Question:** What do I do to enable Telegram communication with agent-researcher?

---

## Current state

The `researcher` agent exists in openclaw.json with the correct workspace:
```
researcher → /home/node/workspace/agents/agent-researcher
```

It has no Telegram binding. All 5 operational board leads have a binding of this form:
```json
{
  "agentId": "lead-6bddb9d2-...",
  "match": {
    "channel": "telegram",
    "peer": { "kind": "group", "id": "-5146184666" }
  }
}
```

You need to add the same for `researcher`.

---

## Steps

### Step 1 — Create a Telegram group for Compass

Create a new Telegram group. Suggested name: **"Compass — IDEA"**

Add `@Idea911Bot` to the group.

### Step 2 — Get the group chat ID

Send any message in the new group. Then open this URL in a browser (or curl it):

```
https://api.telegram.org/botTELEGRAM_BOT_TOKEN_REDACTED/getUpdates
```

Look for the `chat.id` field in the response — it will be a negative number like `-5123456789`. That is the group ID.

### Step 3 — Update openclaw.json

Edit `/var/lib/docker/volumes/openclaw_openclaw-data/_data/openclaw.json`:

**Add a binding** (in the `bindings` array):
```json
{
  "agentId": "researcher",
  "match": {
    "channel": "telegram",
    "peer": { "kind": "group", "id": "<YOUR_GROUP_ID>" }
  }
}
```

**Add the group ID to the allowlist** (in `channels.telegram.allowFrom`):
```json
"<YOUR_GROUP_ID>"
```

### Step 4 — Restart the OpenClaw gateway

```bash
cd /home/pi/openclaw/openclaw && docker compose restart
```

### Step 5 — Test

Send a message in the new Compass group. The bot should route it to the `researcher` agent session.

---

## Notes

- The `researcher` agent uses `dangerouslySkipPermissions: true` (set via `sandbox.mode: "all"` — same as other agents). All tool calls auto-approved.
- The `researcher` agent's workspace is already set to `/home/node/workspace/agents/agent-researcher` in openclaw.json — this was updated earlier in this session.
- Compass's CLAUDE.md, AGENTS.md, SOUL.md, BOOTSTRAP.md, etc. are all in place at that workspace path.
- After the gateway restart, OpenClaw will pick up the new binding immediately.

---

*Compass · Strategic Advisor · IDEA*

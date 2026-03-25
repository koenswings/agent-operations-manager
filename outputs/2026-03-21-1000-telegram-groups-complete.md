> **Question:** I created the groups. What do I do to connect them to each agent? (follow-up: completing the setup)

## Telegram Groups — Setup Complete

All 5 Telegram groups are now connected to their respective agents.

### Group → Agent mapping

| Telegram Group | Chat ID | Agent | Board ID prefix |
|----------------|---------|-------|-----------------|
| IDEA - Axle | `-5146184666` | Axle (Engine Dev) | `lead-6bddb9d2` |
| IDEA - Pixel | `-5187034968` | Pixel (Console Dev) | `lead-ac508766` |
| IDEA - Beacon | `-5139661372` | Beacon (Site Dev) | `lead-7cc2a1cf` |
| IDEA - Veri | `-5286995682` | Veri (Quality Mgr) | `lead-d0cfa49e` |
| IDEA - Marco | `-5141459717` | Marco (Programme Mgr) | `lead-3f1be9c8` |

### What was done

1. **openclaw.json bindings** updated — each group bound to its agent via `peer: { kind: "group", id: "<chat_id>" }` filter
2. **allowFrom** updated — all 5 group chat IDs added alongside the CEO user ID
3. **groupAllowFrom** — set to `["8320646468"]` so only the CEO can trigger agents in groups
4. **requireMention: false** — bot responds to all messages in groups without needing @mention
5. **All 5 agent MEMORY.md files** updated with group name and chat ID
6. **Privacy mode disabled** on @Idea911Bot via @BotFather — required for bot to receive non-command messages in groups

### What changed in openclaw.json

```json
"channels": {
  "telegram": {
    "dmPolicy": "allowlist",
    "groupPolicy": "allowlist",
    "groupAllowFrom": ["8320646468"],
    "allowFrom": ["8320646468", "-5146184666", "-5187034968", "-5139661372", "-5286995682", "-5141459717"],
    "groups": { "*": { "requireMention": false } },
    "streamMode": "partial"
  }
},
"bindings": [
  { "agentId": "lead-6bddb9d2-...", "match": { "channel": "telegram", "peer": { "kind": "group", "id": "-5146184666" } } },
  { "agentId": "lead-ac508766-...", "match": { "channel": "telegram", "peer": { "kind": "group", "id": "-5187034968" } } },
  { "agentId": "lead-7cc2a1cf-...", "match": { "channel": "telegram", "peer": { "kind": "group", "id": "-5139661372" } } },
  { "agentId": "lead-d0cfa49e-...", "match": { "channel": "telegram", "peer": { "kind": "group", "id": "-5286995682" } } },
  { "agentId": "lead-3f1be9c8-...", "match": { "channel": "telegram", "peer": { "kind": "group", "id": "-5141459717" } } }
]
```

### Lessons learned (non-obvious issues encountered)

1. **groupAllowFrom ≠ allowFrom** — Telegram group authorization uses a separate `groupAllowFrom` field, not the top-level `allowFrom`. Without it, `groupPolicy: "allowlist"` silently drops all group messages with no log output at INFO level.
2. **Privacy mode** — Telegram bots have privacy mode ON by default. In this mode, bots only see /commands and @mentions in groups. Must be disabled via @BotFather for the bot to receive plain messages.
3. **peer schema** — The `peer` field in bindings is `{ kind: "group", id: "<chat_id>" }` — not a plain string.
4. **bindings.find()** — OpenClaw routing uses the first matching binding only; all 5 bindings with identical `{ channel: "telegram" }` meant all messages went to Axle only.

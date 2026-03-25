> **Question:** Help me debug Telegram — I think it is not working

## Diagnosis

### What was working
- Bot `@Idea911Bot` (token `8496871029:...`) was live and polling — `channels status` confirmed `enabled, configured, running, mode:polling`
- CEO user ID `8320646468` was in the allowlist and recognised by the gateway (`directory peers list` showed 1 peer)
- Bot token valid, `getMe` returned the bot profile correctly

### Root cause
**No routing bindings existed.** The `bindings` key was absent from `openclaw.json`. When a DM arrives from Telegram, OpenClaw performs a deterministic routing lookup — if no binding matches (no agent claims that channel), the message is silently dropped. Every agent showed `(no channels)`.

### Fix applied
Added a top-level `bindings` array to `/var/lib/docker/volumes/openclaw_openclaw-data/_data/openclaw.json` with 5 entries, one per board lead:

| Agent | ID |
|-------|----|
| Axle (Engine Dev) | `lead-6bddb9d2-c06f-444d-8b18-b517aeaa6aa8` |
| Pixel (Console Dev) | `lead-ac508766-e9e3-48a0-b6a5-54c6ffcdc1a3` |
| Beacon (Site Dev) | `lead-7cc2a1cf-fa22-485f-b842-bb22cb758257` |
| Veri (Quality Mgr) | `lead-d0cfa49e-edcb-4a23-832b-c2ae2c99bf67` |
| Marco (Programme Mgr) | `lead-3f1be9c8-87e7-4a5d-9d3b-99756c35e3a9` |

Each binding: `{ "agentId": "<id>", "match": { "channel": "telegram" } }`

Hot reload triggered and applied automatically (debounceMs: 750). `config get bindings` confirms all 5 are live.

## ⚠️ Behaviour note — broadcast vs single agent

All 5 bindings have identical specificity (`channel: "telegram"` with no peer or group filter). OpenClaw's routing rules say "most-specific wins" — when specificity is equal, the behaviour may be **broadcast** (all 5 agents receive every DM). This needs to be tested.

**If broadcast is too noisy:** narrow the bindings by assigning specific Telegram groups/topics to individual agents, or reduce to a single default agent (Marco) and address others by name within that conversation.

## How to test

1. Open Telegram and DM `@Idea911Bot`
2. Send a message (from account with ID `8320646468`)
3. Check gateway logs: `sudo docker logs -f openclaw-gateway`
4. Look for `[telegram]` routing and agent activity logs

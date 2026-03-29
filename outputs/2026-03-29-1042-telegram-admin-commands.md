> **Task/Question:** Can you confirm that I can also give you a command via this channel to rotate the keys or to add or revoke a user? Please also document this option.

## Confirmation

Yes — Koen can issue admin commands directly via Telegram. Atlas will action them immediately.

## Scope

### Add / Revoke a trusted user
Trusted sender IDs live in `openclaw.json`. Atlas patches them using the `gateway` tool. No SSH required. Gateway restarts automatically.

### Key rotation

| Key type | Atlas can do | Koen must do |
|---|---|---|
| MC API token (per-agent) | Call MC API to generate new token; update agent `.env` | Restart affected container if needed |
| GitHub PAT | Update `.env` files with provided value | Generate new PAT on GitHub.com |
| Telegram bot token | Patch `openclaw.json`; restart gateway | Rotate via BotFather; provide value |
| OpenClaw session key | Patch `openclaw.json` via `gateway` tool | — |

**Docker caveat:** Host-level Docker restarts require SSH by Koen — Atlas cannot execute those from the sandbox.

## Documentation

Added "Telegram-Based Admin Commands" section to `TOOLS.md` with full command reference and examples. Committed to `memory/updates`.

> **Task/Question:** Write MIGRATE.md and the unified platform compose. Ensure migration is safe given OpenClaw-dependent communication channel.

## What was built

### `idea/platform/compose.yaml`
Unified compose for all 6 IDEA platform services on `idea-net`:
- `openclaw-gateway` — `ghcr.io/openclaw/openclaw:latest`, port 18789, workspace mount, Docker secrets, custom entrypoint
- `mission-control-backend` — `openclaw-mission-control-backend`, port 8000, `BASE_URL=http://openclaw-gateway:18790` (service name replaces host bridge IP)
- `mission-control-frontend` — `openclaw-mission-control-frontend`, port 4000
- `mission-control-db` — `postgres:16-alpine`, volume `mc-db-data`
- `mission-control-redis` — `redis:7-alpine`, volume `mc-redis-data`
- `mission-control-webhook-worker` — `openclaw-mission-control-webhook-worker`

### `idea/scripts/MIGRATE.md`
8-step runbook for Claude Code executing in tmux SSH session:
0. Discovery — verify compose template against actual MC compose file
1. Backup current compose files + config + secrets list
2. Prepare secrets in `idea/platform/secrets/` (copy from `/home/pi/openclaw/secrets/`, extract MC DB password)
3. Validate: `docker compose -f platform/compose.yaml config`
4. Stop standalone MC (OpenClaw stays up — Telegram unaffected)
5. Start unified compose (brief OpenClaw restart, 30–60s downtime)
6. Verify all 6 containers + health checks
7. Update any hardcoded bridge IP references
8. Send confirmation to Atlas via Telegram

Rollback: `docker compose up -d` from original compose dirs (no volumes deleted).

### `idea/scripts/apply-config.sh`
Lightweight script for future config updates: fills token, merges gateway token, writes to `~/.openclaw/openclaw.json`, restarts gateway.

### `idea/platform/` (renamed from `idea/openclaw/`)
Same content, better name reflecting scope (OpenClaw + MC).

## Key technical discoveries (from docker inspect)
- OpenClaw uses pre-built image (no source build needed)
- Credentials are Docker secrets (bind-mounted files), not env vars
- Custom entrypoint.sh from `/home/pi/openclaw/entrypoint.sh`
- MC has 5 containers, currently on separate network (`172.19.0.0/24`)
- MC DB password is in the MC compose file (not captured here — Claude Code reads it during migration)

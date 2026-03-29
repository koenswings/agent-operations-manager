> **Task/Question:** Check if all is OK after the Pi crash during the IDEA Platform migration. Verify migration completed correctly and confirm current state.

## Summary

The unified compose migration completed successfully. All 6 services are running on the IDEA Platform (`idea/platform/compose.yaml`). One issue was found and fixed post-migration: stale `AUTH_TOKEN` in all 5 agent `.env` files.

---

## Migration Recap

### What was migrated

**Before:** two independent Docker Compose stacks
- `/home/pi/openclaw/` — OpenClaw gateway
- `/home/pi/openclaw/mission-control/` — Mission Control (5 containers)

**After:** single unified stack
- `/home/pi/idea/platform/compose.yaml` — 6 services on shared `idea-net` network

### What Claude Code did (tmux session, `pi@openclaw-pi`)

Steps completed before the Pi crash:
- **Step 1 (Backup):** `pre-migration-20260327-1654/` created at `/home/pi/backups/`
- **Step 2 (Secrets):** `/home/pi/idea/platform/secrets/` created; 3 secret files written

Steps completed in the post-crash recovery session:
- **Step 3 (Validate):** `docker compose -f platform/compose.yaml config` — clean ✅
- **Step 4 (Stop standalone MC):** `docker compose.yml` on the old standalone MC had a YAML error (duplicate `BASE_URL` key at line 52). Stopped by project name (`docker compose -p openclaw-mission-control down`) then directly (`docker stop <containers>`).
- **Step 5 (Start unified compose):** `sudo docker compose -f platform/compose.yaml up -d` — all 6 containers created and started ✅
- **Step 6 (Verify):** All services healthy; `curl http://localhost:18789/health` returned `{"ok":true}`; MC API `{"ok":true}`; MC frontend HTTP 200 ✅
- **Step 7 (Check stale IPs):** No `172.18.0` references in `openclaw.json` — no update needed ✅

---

## Post-Migration Health Check (Atlas)

Performed 2026-03-27 ~18:00 UTC.

### Container status

| Container | Status |
|---|---|
| `openclaw-gateway` | ✅ Up, healthy |
| `openclaw-mission-control-backend-1` | ✅ Up |
| `openclaw-mission-control-frontend-1` | ✅ Up |
| `openclaw-mission-control-db-1` | ✅ Up, healthy |
| `openclaw-mission-control-redis-1` | ✅ Up, healthy |
| `openclaw-mission-control-webhook-worker-1` | ✅ Up |

### MC database

- PostgreSQL volume (`openclaw-mission-control_postgres_data`) reused — no data loss
- All 5 boards intact: Engine Dev, Console Dev, Site Dev, Programme Manager, Quality Manager
- IDEA org intact

### Issue found and fixed: stale AUTH_TOKEN

**Root cause:** MC backend was started with `AUTH_MODE=local` using `MC_LOCAL_AUTH_TOKEN` from `platform/.env`. This is a new credential, different from the per-agent tokens that were in use with the old standalone MC.

**Effect:** All 5 agent `.env` files had stale `AUTH_TOKEN` values that the new MC backend rejected with `401 Unauthorized`.

**Fix:** Updated `AUTH_TOKEN` in all 5 `.env` files to the current `MC_LOCAL_AUTH_TOKEN` value from `platform/.env`. Note: the token contains `+` and `=` characters that get mangled in shell variable expansion — use Python `urllib` or export carefully.

**Files updated (gitignored, not committed):**
- `agents/agent-operations-manager/.env`
- `agents/agent-engine-dev/.env`
- `agents/agent-console-dev/.env`
- `agents/agent-site-dev/.env`
- `agents/agent-programme-manager/.env`

### Networking

- All 6 containers are on `idea-net` (Docker bridge)
- `mission-control-backend` resolves correctly via Docker DNS from `openclaw-gateway` → `172.20.0.5`
- The old `172.18.0.1:8000` bridge IP also still routes (port 8000 published to host)
- Both URLs work; service name is preferred for reliability

---

## Follow-up actions

### Done in this session
- [x] AUTH_TOKEN updated in all 5 agent `.env` files
- [x] MC API verified: all 5 boards accessible with new token

### Pending (PR)
- [ ] Update `BASE_URL` in all 5 agent `TOOLS.md` files: `172.18.0.1:8000` → `http://mission-control-backend:8000`
- [ ] Update `skills/mc-api/SKILL.md` and `scripts/mc-refresh.sh`
- [ ] Update `design/virtual-company-design.md`: SQLite→PostgreSQL, MC ports, unified compose note

### Post-migration cleanup (Koen, after 24h+ stable operation)
- Old `/home/pi/openclaw/` directory: keep as reference but do not `docker compose up` from it again
- The old standalone MC compose file has a YAML error (duplicate BASE_URL) — can be ignored since it's no longer in use

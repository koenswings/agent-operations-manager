> **Question:** Re-assess the Tailscale key leak, audit all repos for other leaked keys (especially Anthropic), and implement the Tailscale key as a secret file instead of hardcoded in compose.

---

## Security Incident Assessment

### What likely happened

1. The Tailscale pre-auth key was committed to `koenswings/app-openclaw` on Feb 21 2026 (GitGuardian detected this)
2. The key allowed a third party to join your tailnet — or more plausibly, simply read the Anthropic key from the compose file or environment
3. Your Anthropic credit was drained by someone using the leaked API key
4. You have since revoked both keys and regenerated the Anthropic key ✓

**Root cause:** TS_AUTHKEY was passed as an env var (`${ts_authkey}`) in the compose file, which was then populated from a `.env` file. Either the `.env` file was accidentally committed, or the key value was briefly hardcoded in the compose. Either way, Tailscale access to your Pi is plausible.

---

## Full Secrets Audit — All Repos

### Clean (no secrets in git)

| Location | Status |
|----------|--------|
| `/home/pi/openclaw/.gitignore` | `secrets/` is gitignored ✓ |
| `/home/pi/openclaw/secrets/anthropic_api_key.txt` | Not tracked, permissions 600 ✓ |
| `/home/pi/openclaw/secrets/openclaw_gateway_token.txt` | Not tracked, permissions 600 ✓ |
| `/home/pi/openclaw/mission-control/.env` | Not tracked, gitignored ✓ |
| `/home/pi/openclaw/mission-control/backend/.env.test` | Tracked but contains only placeholder test token ✓ |
| `app-disk/instances/openclaw-dev/.env.template` | Empty value (`ts_authkey=`), no real key ✓ |
| Git history (openclaw) | 2 commits, no secrets in tracked files ✓ |

### Agent workspace files — AUTH_TOKENs in TOOLS.md / HEARTBEAT.md

The security scan found agent AUTH_TOKENs hardcoded in workspace files:

| Workspace | Files | Notes |
|-----------|-------|-------|
| `workspace-gateway-ccc8463b...` | AGENTS.md, TOOLS.md, HEARTBEAT.md | MC auth token |
| `workspace-lead-6bddb9d2...` (Axle) | TOOLS.md, HEARTBEAT.md | MC auth token |
| `workspace-lead-3f1be9c8...` (Marco) | TOOLS.md, HEARTBEAT.md | MC auth token |
| `workspace-lead-7cc2a1cf...` (Beacon) | TOOLS.md, HEARTBEAT.md | MC auth token |
| `workspace-lead-ac508766...` (Pixel) | TOOLS.md, HEARTBEAT.md | MC auth token |
| `workspace-lead-d0cfa49e...` (Veri) | TOOLS.md, HEARTBEAT.md | MC auth token |

**Risk level:** Medium. These are Mission Control agent tokens (not Anthropic keys), used in HTTP headers for agent ↔ MC communication. They are inside `/home/pi/idea/` which is not a git repo tracked on GitHub. As long as the idea directory is not pushed to a public repo, exposure risk is low — but anyone with filesystem access (e.g. via a compromised Tailscale node) could read these.

**Action recommended:** After this incident is resolved, rotate these tokens by reprovisioning the agents in Mission Control.

---

## Code Change Implemented — Tailscale Key via Docker Secret

### What changed

The `TS_AUTHKEY` env var has been removed from both compose files. Instead, the Tailscale container now reads the key from a Docker secret file, using the same pattern as the Anthropic and gateway token secrets.

**Files modified:**
- `app-disk/instances/openclaw-dev/compose.yaml`
- `app-disk/apps/openclaw-1.0.0/compose.yaml`
- `app-disk/instances/openclaw-dev/.env.template` — removed `ts_authkey=` line
- `app-disk/instances/openclaw-dev/secrets/README.md` — added `tailscale_authkey.txt` instructions

**New files created:**
- `app-disk/instances/openclaw-dev/tailscale-entrypoint.sh`
- `app-disk/apps/openclaw-1.0.0/tailscale-entrypoint.sh`

### How it works

```sh
# tailscale-entrypoint.sh
#!/bin/sh
set -e
if [ -f "/run/secrets/tailscale_authkey" ]; then
  export TS_AUTHKEY=$(cat /run/secrets/tailscale_authkey | tr -d '\n')
fi
exec /usr/local/bin/containerboot "$@"
```

The tailscale service in compose now:
1. Mounts `tailscale-entrypoint.sh` into the container
2. Overrides the entrypoint to run the wrapper script
3. Mounts `secrets/tailscale_authkey.txt` as a Docker secret at `/run/secrets/tailscale_authkey`
4. The wrapper reads the file and sets `TS_AUTHKEY` before handing off to `containerboot`

### To use

Place your new Tailscale pre-auth key in the secrets file before starting the stack:

```bash
echo -n 'tskey-auth-...' > app-disk/instances/openclaw-dev/secrets/tailscale_authkey.txt
chmod 600 app-disk/instances/openclaw-dev/secrets/tailscale_authkey.txt
```

The file is gitignored via the `secrets/` rule. The key will never appear in any compose or env file.

### The production `/home/pi/openclaw/compose.yaml`

This file was NOT changed because it does not use `TS_AUTHKEY` at all — the Tailscale state is already persisted in a volume (`tailscale-state`) from the initial auth. No re-auth key is needed for normal restarts.

---

## Immediate Actions Required

| # | Action | Status |
|---|--------|--------|
| 1 | Revoke leaked Tailscale key | ✓ Done |
| 2 | Revoke leaked Anthropic key | ✓ Done |
| 3 | Regenerate Anthropic key and store in secrets file | ✓ Done |
| 4 | Check Tailscale machine list for unauthorized devices | **Pending — do this now** |
| 5 | Move TS_AUTHKEY to Docker secret file (code change) | ✓ Done (this session) |
| 6 | Commit and push changes to `app-openclaw` repo | **Pending** |
| 7 | Rotate Mission Control agent AUTH_TOKENs | Recommended — medium priority |

---

## Commit and Push

The changes to `/home/pi/openclaw` are staged and ready to commit. To push:

```bash
cd /home/pi/openclaw
git add app-disk/apps/openclaw-1.0.0/compose.yaml \
        app-disk/apps/openclaw-1.0.0/tailscale-entrypoint.sh \
        app-disk/instances/openclaw-dev/compose.yaml \
        app-disk/instances/openclaw-dev/.env.template \
        app-disk/instances/openclaw-dev/secrets/README.md \
        app-disk/instances/openclaw-dev/tailscale-entrypoint.sh
git commit -m "Move Tailscale auth key to Docker secret file"
git push
```

---

## Summary

The most likely attack vector: the Tailscale key let someone join your tailnet, gain local network access to the Pi, and read the Anthropic key from a running container's environment or from a local file. Both keys are now revoked and regenerated. The code change ensures the Tailscale auth key can never be accidentally committed — it lives only in `secrets/tailscale_authkey.txt`, which is gitignored, just like the Anthropic key.

> **Question:** On the url: timed out during opening handshake

## Root Cause

Two separate issues blocked the MC backend from connecting to the OpenClaw gateway.

### Issue 1 — `ssl=None` incompatibility (already fixed in previous session)
`websockets` v16 raised `ValueError: ssl=None is incompatible with a wss:// URI`.
Fix: `gateway_rpc.py` no longer passes `ssl=None` — the `ssl` kwarg is only added to
`connect_kwargs` when a custom (insecure) context is explicitly needed.
**Status:** Already applied before this session.

### Issue 2 — Tailscale IP not reachable from Docker (fixed this session)

`compose.yml` had:
```yaml
extra_hosts:
  - "openclaw-pi.tail2d60.ts.net:100.102.15.46"
```

`100.102.15.46` is the Tailscale IP — but it is bound to the `tailscale0` TUN interface
**inside the Tailscale container's own network namespace**. Other Docker containers
(including the MC backend) run in isolated namespaces and cannot route to this IP.
Result: every connection attempt to `wss://openclaw-pi.tail2d60.ts.net:443` timed out.

## Fix

**`/home/pi/openclaw/mission-control/compose.yml`** — both `backend` and `webhook-worker`:

```yaml
# Before
extra_hosts:
  - "openclaw-pi.tail2d60.ts.net:100.102.15.46"

# After
extra_hosts:
  - "openclaw-pi.tail2d60.ts.net:host-gateway"
```

`host-gateway` is a Docker Compose special value that resolves at container start to the
container's default gateway IP. For MC containers on the `openclaw-mission-control_default`
bridge (`172.19.0.0/16`), this is `172.19.0.1` — the Pi host on that bridge.

Port 18789 is confirmed reachable at `172.19.0.1:18789` (gateway publishes `0.0.0.0:18789`).

## Gateway URL for Step 14

When registering the gateway in MC UI, use:

```
ws://openclaw-pi.tail2d60.ts.net:18789
```

**Not** `wss://openclaw-pi.tail2d60.ts.net:443`. Tailscale provides TLS only for
external browser access; the MC backend connects via the Docker host bridge and does
not need SSL. The gateway speaks plain WebSocket on port 18789.

## Second error: gateway reports `version: "dev"`

After fixing the network, the MC backend connected successfully but the version check
failed: "Gateway reported an unsupported version format 'dev'. Minimum supported version
is 2026.02.9."

The gateway CLI sets its server version from:
```javascript
version: process.env.OPENCLAW_VERSION ?? process.env.npm_package_version ?? "dev"
```
Neither var was set → `"dev"`. The actual version (from `/app/build-info.json`) is
`2026.2.17`.

**Fix — `/home/pi/openclaw/compose.yaml`:**
```yaml
environment:
  - OPENCLAW_VERSION=2026.2.17   # added
```

## Result

Both stacks restarted. Gateway registration succeeded:

```json
{
  "id": "ccc8463b-8b7d-46a1-afea-f699081b44c2",
  "name": "OpenClaw Pi",
  "url": "ws://openclaw-pi.tail2d60.ts.net:18789",
  "workspace_root": "/home/node/workspace"
}
```

MC backend can now reach and provision agents on the gateway. Step 14 is partially
complete — gateway is registered. Next: create boards and pair browser (live session).

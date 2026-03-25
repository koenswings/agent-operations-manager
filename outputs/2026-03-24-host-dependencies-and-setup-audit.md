# Host Dependencies & Reproducibility Audit
_2026-03-24 — Compass_

---

## 1. What was changed / added outside of git

### A. Inside the OpenClaw Docker container (ephemeral — lost on container rebuild)

| Dependency | How installed | Why needed |
|---|---|---|
| `chromium` | `apt-get install -y chromium` | md-to-pdf skill: headless PDF rendering |
| `chromium-common` | pulled in by chromium | (dependency) |
| `chromium-sandbox` | pulled in by chromium | (dependency) |
| `node_modules` symlink | `ln -s .../agent-engine-dev/node_modules .../skills/md-to-pdf/scripts/node_modules` | md-to-pdf skill: resolves `zx` and `markdown-it` from engine-dev's node_modules |

These are NOT in the OpenClaw image. They exist only in the running container's filesystem.
If the container is rebuilt or the image is re-pulled, they are gone.

### B. In each agent workspace (.env files — gitignored, must be recreated)

| File | Content | Why needed |
|---|---|---|
| `agents/agent-*/env` | `GITHUB_TOKEN=ghp_...` | Agents can push branches and open PRs |

### C. In openclaw.json (in Docker volume — captured in app-openclaw scrubbed template)

| Change | Value |
|---|---|
| `skills.load.extraDirs` | `["/home/node/workspace/skills"]` |

### D. Pi host filesystem (persists; captured in git after today's session)

| Item | Location | Status |
|---|---|---|
| Org root git repo | `/home/pi/idea/` | ✅ Initialized and pushed to `koenswings/idea` |
| Shared skills | `/home/pi/idea/skills/` | ✅ In git |
| Org root files | `CONTEXT.md`, `ROLES.md`, `PROCESS.md`, etc. | ✅ In git |

---

## 2. Non-reproducible risks (ranked by severity)

| Risk | Impact | Fix |
|---|---|---|
| Chromium not in container image | md-to-pdf skill broken after rebuild | Add to OpenClaw Dockerfile or setup.sh |
| `node_modules` symlink missing | md-to-pdf skill broken after fresh clone | Add creation step to setup.sh |
| Agent `.env` files missing | Agents can't push to GitHub | Document in setup.sh; secrets added manually |
| MC auth token not documented | Can't access MC from scripts after rebuild | Already stored at `/home/pi/.mc-token` — ensure setup.sh documents this |

---

## 3. Task for Axle's board

**Title:** Add host dependencies to `idea/scripts/setup.sh`

**Description:**
Two dependencies were added to the running OpenClaw container that are not captured
in any setup script. If the container is rebuilt, the `md-to-pdf` skill will break silently.

**Dependencies to add:**

1. **Chromium** — installed in the OpenClaw container for headless PDF rendering:
   ```bash
   apt-get install -y chromium
   ```

2. **node_modules symlink** — links the shared skill to engine-dev's Node.js dependencies:
   ```bash
   ln -s /home/node/workspace/agents/agent-engine-dev/node_modules \
     /home/node/workspace/skills/md-to-pdf/scripts/node_modules
   ```

These steps should be added to `idea/scripts/setup.sh` in the section that runs
post-deployment configuration inside the OpenClaw container (or as a `docker exec` call).

The setup.sh is not yet written (it's a backlog item). This task should either:
- Block on setup.sh being created first and then add these as steps, OR
- Document these steps in a `README.md` or `INSTALL.md` in `idea/openclaw/` as an interim measure.

**Tags:** infrastructure, reproducibility
**Priority:** medium (system works now; only breaks after a container rebuild)

---

## 4. Summary: installing the IDEA virtual company on a fresh Pi

_Excludes OpenClaw platform deployment (separate concern)._

### Prerequisites
- Raspberry Pi 5 with Raspberry Pi OS (Debian 12 Bookworm)
- Docker installed
- Tailscale installed and authenticated
- OpenClaw + Mission Control already running (app-openclaw stack)
- Git installed

### Step 1 — Clone the org root and all agent repos

```bash
cd /home/pi
git clone https://github.com/koenswings/idea.git idea
cd idea/agents
git clone https://github.com/koenswings/agent-engine-dev.git
git clone https://github.com/koenswings/agent-console-dev.git
git clone https://github.com/koenswings/agent-site-dev.git
git clone https://github.com/koenswings/agent-quality-manager.git
git clone https://github.com/koenswings/agent-programme-manager.git
git clone https://github.com/koenswings/agent-researcher.git
```

### Step 2 — Install agent npm dependencies (engine-dev)

```bash
cd /home/pi/idea/agents/agent-engine-dev
pnpm install
```

### Step 3 — Create node_modules symlink for shared skills

```bash
ln -s /home/node/workspace/agents/agent-engine-dev/node_modules \
  /home/node/workspace/skills/md-to-pdf/scripts/node_modules
```
_(Run this inside the OpenClaw container, or via `docker exec openclaw bash -c "..."`)_

### Step 4 — Install chromium in the OpenClaw container

```bash
docker exec -it openclaw apt-get install -y chromium
```

### Step 5 — Add GITHUB_TOKEN to each agent workspace

```bash
echo "GITHUB_TOKEN=ghp_YOUR_TOKEN_HERE" >> /home/pi/idea/agents/agent-engine-dev/.env
echo "GITHUB_TOKEN=ghp_YOUR_TOKEN_HERE" >> /home/pi/idea/agents/agent-console-dev/.env
echo "GITHUB_TOKEN=ghp_YOUR_TOKEN_HERE" >> /home/pi/idea/agents/agent-site-dev/.env
echo "GITHUB_TOKEN=ghp_YOUR_TOKEN_HERE" >> /home/pi/idea/agents/agent-quality-manager/.env
echo "GITHUB_TOKEN=ghp_YOUR_TOKEN_HERE" >> /home/pi/idea/agents/agent-programme-manager/.env
echo "GITHUB_TOKEN=ghp_YOUR_TOKEN_HERE" >> /home/pi/idea/agents/agent-researcher/.env
```

### Step 6 — Apply openclaw.json config

Copy the template from `app-openclaw/openclaw.json` into the OpenClaw Docker volume,
fill in real values (Telegram bot token, gateway token), and restart:

```bash
# Copy to Docker volume
docker cp /home/pi/openclaw/openclaw.json openclaw:/root/.openclaw/openclaw.json
docker restart openclaw
```

### Step 7 — Configure Mission Control

- Create the IDEA org in MC
- Create agent boards: Engineering (Axle, Pixel, Veri), HQ (Marco, Compass)
- Store the MC auth token at `/home/pi/.mc-token`

### Step 8 — Run BOOTSTRAP sessions for all new agents

Open each agent's Telegram group and say hello. Each agent will introduce itself and
complete its BOOTSTRAP.md.

---

## 5. What's missing / still needs doing

- `idea/scripts/setup.sh` — this audit is a first draft of its content; the actual script doesn't exist yet
- `idea/openclaw/` folder — scrubbed `openclaw.json` template to copy on fresh install
- MC board setup script — currently done manually via the MC UI
- Container name for `docker exec` above — verify the actual container name on the running system

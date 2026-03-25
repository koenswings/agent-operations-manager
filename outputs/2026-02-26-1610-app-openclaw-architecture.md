> **Question:** I would like the openclaw and mission control web app combination with tailscale to be modelled as an app disk for the project that, when instantiated on a pi turns that Pi into a full-fledged ai-assisted development machine. This means that the whole workspace needs to be in the data folder of that app and hence the initial config that we are preparing here will go into the init_data.tar.gz on the app-openclaw repo. Unless we say that this is too specific and that we should keep the app-openclaw more generic so it can be used for many more things. But how do we deploy this? How do we explain to an engineer how to turn his pi into a full-fledged ai-assisted dev machine using the app-openclaw app disk? He has to create an app disk, and then needs to clone the data for the idea organisation into it... Can you advise me a bit how to proceed?

# app-openclaw Architecture: Generic Platform vs IDEA Config

## The core tension

You have identified a real architectural decision: bake IDEA-specific config into the
App Disk, or keep `app-openclaw` generic and layer IDEA config on top separately.

The answer becomes clear when you look at what actually lives in each layer.

---

## Three distinct layers

### Layer 1 — Platform (generic, reusable)
What OpenClaw needs to run anywhere:
- Docker Compose for OpenClaw gateway + Mission Control
- Empty `openclaw.json` template (no agents, no tokens)
- Tailscale installation and activation hook
- Startup scripts and health checks

This is `app-openclaw`. It is organisation-agnostic. Someone running a software
consultancy, a school network, or a research lab could use the same App Disk.
`init_data.tar.gz` contains this layer only.

### Layer 2 — Organisation config (IDEA-specific)
What makes the Pi an IDEA development machine:
- `openclaw.json` with the six IDEA agents, workspaces, heartbeats, channel policies
- The `idea/` directory structure (CONTEXT.md, ROLES.md, PROCESS.md, agent subfolders)
- `AGENTS.md` files for each agent
- Mission Control board hierarchy

This layer lives in the `idea` GitHub repo (in `idea/openclaw/` as discussed), not in
the App Disk. It is pulled in at setup time, not baked into the disk image.

### Layer 3 — Live data (runtime only, never in git)
- Cloned agent repos with working trees and git history
- Mission Control SQLite database (tasks, activity timeline)
- WhatsApp session pairing state
- Tailscale machine identity and auth
- Any credentials or tokens

This layer is created when the engineer runs setup and as the system operates.
It lives on the data partition of the App Disk and is never committed anywhere.

---

## The setup story for a new IDEA engineer

With this separation, the engineer experience becomes:

**Step 1 — Create and dock the App Disk**
Create an App Disk from `app-openclaw`. Dock it. OpenClaw, Mission Control, and
Tailscale start up — empty, no agents, but fully functional platform.

**Step 2 — Run the IDEA setup script**
```bash
curl -fsSL https://raw.githubusercontent.com/idea-edu-africa/idea/main/scripts/setup.sh | bash
```
Or if they already have git access:
```bash
git clone https://github.com/idea-edu-africa/idea /tmp/idea-setup
/tmp/idea-setup/scripts/setup.sh
```
The setup script:
1. Creates the workspace directory structure under the App Disk data folder
2. Clones `idea` and all six agent repos into `agents/`
3. Applies `idea/openclaw/openclaw.json` to the live OpenClaw instance
4. Configures Mission Control board hierarchy via its API

**Step 3 — Enter credentials (manual, once)**
- Tailscale: `tailscale up --auth-key=<key>`
- WhatsApp: scan QR code with dedicated SIM phone
- Mission Control bearer token: set `LOCAL_AUTH_TOKEN` in openclaw.json, restart

**Step 4 — Bootstrap agent sessions**
Open each agent tab in Mission Control, run the BOOTSTRAP session to confirm
identity and orientation. The Pi is now a fully configured IDEA development machine.

---

## What goes where

| Content | Location | Reason |
|---------|----------|--------|
| OpenClaw + Mission Control Docker setup | `app-openclaw` repo → `init_data.tar.gz` | Generic platform |
| Tailscale install hook | `app-openclaw` repo → `init_data.tar.gz` | Generic platform |
| IDEA agent roster (`openclaw.json`, no tokens) | `idea` repo → `idea/openclaw/` | Org config, version-controlled |
| IDEA workspace structure (`AGENTS.md` files etc.) | `idea` repo + agent repos | Org content |
| Setup script | `idea` repo → `idea/scripts/setup.sh` | Ties it together |
| Credentials, tokens, pairing state | Live on disk only, never committed | Secrets |
| Mission Control database, agent memory | Live on disk only | Runtime state |

---

## Why not bake IDEA config into init_data?

Two reasons:

1. **It conflates two different things.** The App Disk answers "how do I run OpenClaw?"
   The `idea` repo answers "how is IDEA configured?" Mixing them means every IDEA org
   config change requires a new App Disk image — that's the wrong granularity.

2. **It breaks for the second engineer.** The first engineer creates the disk from scratch.
   The second engineer should be able to dock a fresh `app-openclaw` disk and pull the
   current IDEA config from GitHub — not from a potentially stale disk image. The `idea`
   repo is always current; a baked init_data is a snapshot.

---

## The right mental model

> `app-openclaw` is the runtime. `idea` is the configuration. The setup script is the
> installer that wires them together on a specific machine.

This is the same pattern as: Docker is the runtime, your compose.yaml is the config,
and `docker compose up` is the installer.

---

## One open question

Is this Pi intended only for IDEA internal use (you, maybe one or two contributors),
or is the eventual goal to give partner organisations their own AI-assisted dev machine
running their own configuration?

If IDEA-internal only: the semi-manual setup above is perfectly adequate.

If the goal is to distribute to others: the setup script would need to be parameterised
(pass in a GitHub org, a config repo URL, an API key for the LLM provider) and the
whole thing becomes a more general "AI dev machine provisioner" with `app-openclaw`
as the runtime and a per-organisation config repo as the input. Worth designing for
now if distribution is on the horizon.

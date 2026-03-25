> **Question:** Shouldn't we restructure the openclaw repo into the app-openclaw repo that follows the template of apps on the solution and allows openclaw to be instantiated on an app disk. See the engine project for more information on apps and how they are stored in github repos and instantiated on app disks.

# Restructuring openclaw as app-openclaw

## What the Engine's app model requires

From reading the engine project, every app repo must follow this structure:

```
app-<name>/
├── compose.yaml        ← REQUIRED, must include x-app metadata block
├── init_data.tar.gz    ← OPTIONAL, extracted to data/ on first instantiation
└── README.md
```

The `x-app` block in `compose.yaml` is parsed by the Engine on disk insertion:

```yaml
x-app:
  name: openclaw
  version: "1.0.0"
  title: "OpenClaw AI Platform"
  description: "OpenClaw + Mission Control + Tailscale"
  url: "/"
  category: "Utilities"
  icon: "<icon-url>"
  author: "IDEA"
```

The build script `build-app-instance.ts` clones the app repo from GitHub
(`https://github.com/<org>/app-<name>`), provisions the disk layout, and the Engine
auto-instantiates on dock.

---

## The structural fit is good — but there is a meaningful difference

School apps (Kolibri, Nextcloud, Wikipedia) run FROM the App Disk:
- Services start when disk is docked, stop when undocked
- All data lives on the USB disk
- No internet required after provisioning

OpenClaw as a development machine is different:
- It must run persistently, independent of any disk being docked
- It connects to external services (Anthropic API, WhatsApp, Tailscale) — internet required
- Its data (agent memory, Mission Control SQLite, WhatsApp session) needs to survive reboots
- It is the management layer for the Engine, not a field-deployed app on it

**OpenClaw cannot be a standard Engine app** — if it ran from a USB disk, it would stop
every time that disk was undocked. It needs to be permanently installed to the Pi's local
storage.

---

## The right model: "installation app" pattern

The Engine currently has no formal concept of an "installation app" — a disk that
provisions the Pi and installs services to local storage rather than running from the disk.
That is a potential future Engine feature.

For now, the pragmatic approach is:

**Follow the app-* template exactly** (same repo structure, same compose.yaml x-app
metadata, same init_data.tar.gz layout), but provision manually via `setup.sh` rather
than via the Engine's auto-instantiation. This gives you:

1. Consistency — anyone familiar with the app-* pattern immediately understands the repo
2. Forward compatibility — when the Engine gains "installation app" support, app-openclaw
   is already in the right shape
3. Discoverability — the repo sits naturally alongside app-kolibri, app-nextcloud, etc. in
   the GitHub org

---

## Recommended repo structure for app-openclaw

```
app-openclaw/
├── compose.yaml              ← OpenClaw + Mission Control + Tailscale services
│                               with x-app metadata block
├── init_data.tar.gz          ← Minimal platform bootstrap data:
│                               empty openclaw.json template, startup hooks
│                               NO IDEA-specific config (see idea/openclaw/)
└── README.md                 ← How to provision a Pi using this app:
                                references idea/scripts/setup.sh for org config
```

The `compose.yaml` is the source of truth for the service configuration. `init_data`
provides just enough to get a blank OpenClaw running — the IDEA workspace is layered
on top by `setup.sh` as previously discussed.

---

## What to do with the current openclaw repo

The current `koenswings/openclaw` repo holds the OpenClaw platform config as it exists
today. The migration path:

1. Rename repo to `app-openclaw` on GitHub
2. Transfer to `idea-edu-africa/app-openclaw`
3. Add `x-app` metadata block to `compose.yaml`
4. Structure or create `init_data.tar.gz` with platform-only bootstrap content
5. Update README to describe the installation flow

History is preserved (rename keeps all commits). The `openclaw` name in the Project
Repositories table was already a placeholder — this replaces it cleanly.

---

## Updated Project Repositories table

| Repo | Purpose | Notes |
|------|---------|-------|
| `idea` | Org root: coordination, agents, backlog | |
| `agent-engine-dev` | Engine developer workspace | Renamed from `engine` |
| `agent-console-dev` | Console UI developer workspace | New |
| `agent-site-dev` | Website developer workspace | New |
| `agent-quality-manager` | Quality manager workspace | New |
| `agent-programme-manager` | Programme manager workspace | New |
| `agent-researcher` | Compass — strategic advisor | Renamed from `idea-proposal` |
| `app-openclaw` | OpenClaw + Mission Control + Tailscale App Disk | Renamed from `openclaw` |

Total: **8 repos**. The separate `openclaw` entry is replaced by `app-openclaw` —
no net addition.

---

## Summary

Yes — restructure the openclaw repo into app-openclaw following the app-* template.
It will not auto-instantiate via the Engine (OpenClaw is a Pi-resident service, not a
dockable app), but the template gives structural consistency, forward compatibility, and
natural placement in the GitHub org. The rename is low-cost; the repo history carries over.

The proposal's Project Repositories table and backlog should be updated to replace
`openclaw` with `app-openclaw`.

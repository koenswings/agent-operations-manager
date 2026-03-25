> **Question:** Why can't we mount the code repo of a dev agent as its workspace in the Docker container?

---

## Short answer

You can. The pre-configured agents already do exactly this. The board lead agents ended up with separate workspace dirs as a side-effect of how Mission Control provisions them — not because of a technical requirement. There is one meaningful constraint to be aware of (sandbox scope), but it is not a blocker.

---

## What actually happens today

There are two generations of agent config on this system:

**Pre-configured agents** (defined in `agent-researcher/openclaw.json`, likely also in the Docker volume config):

| Agent ID | Workspace in openclaw.json |
|----------|---------------------------|
| `engine-dev` | `/home/node/workspace/agents/agent-engine-dev` ← the code repo |
| `console-dev` | `/home/node/workspace/agents/agent-console-dev` |
| `site-dev` | `/home/node/workspace/agents/agent-site-dev` |
| `researcher` | `/home/node/workspace/agents/agent-researcher` |

**Board lead agents** (provisioned via Mission Control, defined in the Docker volume):

| Agent ID | Workspace |
|----------|-----------|
| `lead-6bddb9d2-...` (Axle) | `/home/node/workspace/workspace-lead-6bddb9d2-...` ← separate dir |
| All other board leads | Same pattern |

The board lead agents ended up with separate workspace dirs because that is what Mission Control created when it provisioned them. It generated a new directory, populated it with identity files, and set that as the workspace. It did not use the existing code repo dirs.

This was not a deliberate design decision on your part — it was MC's default provisioning behaviour.

---

## The one real constraint: sandbox scope

The pre-configured agents have this in their config:

```json
"sandbox": { "mode": "all", "scope": "agent" }
```

`scope: agent` means the agent's filesystem access is scoped to its workspace directory. If the workspace is the code repo, the agent can only access the code repo.

For a dev agent this is a problem: Axle needs to read org-level files — `CONTEXT.md`, `ROLES.md`, `BACKLOG.md` — which live two levels up at the org root. If the workspace is `agents/agent-engine-dev/` and the sandbox scope is `agent`, those files are outside the sandbox and inaccessible.

The board lead agents have no `sandbox` key at all — they run unrestricted, which is why Axle can navigate freely across the mounted filesystem.

**The fix:** If you pointed a board lead's workspace at the code repo, you would leave the sandbox unconfigured (as it is now for board leads). No sandbox = access to the full mounted volume (`/home/node/workspace/` = `/home/pi/idea/`). The agent starts in the code repo dir but can still reach org files.

---

## What the merged model would look like

If you consolidated to use the code repo as workspace:

```json
{
  "id": "lead-6bddb9d2-...",
  "name": "Axle",
  "workspace": "/home/node/workspace/agents/agent-engine-dev",
  "heartbeat": { "every": "0m", ... }
}
```

The agent starts in the code repo. To make this work you would need:

1. **Identity files in the code repo** — SOUL.md, AGENTS.md, HEARTBEAT.md, TOOLS.md, IDENTITY.md, USER.md moved into `agents/agent-engine-dev/`
2. **Memory files gitignored** — `memory/` and `MEMORY.md` added to `.gitignore` so they do not pollute code commits
3. **SOUL.md and AGENTS.md either committed or gitignored** — they define the agent's role so committing them makes sense; they would appear in the repo as documentation of who the agent is and how it works

The workspace-lead directory would no longer be needed for that agent.

---

## Why the separation exists at all — the honest assessment

The separate workspace dirs are an artefact of Mission Control's provisioning flow, not a deliberate architectural choice. MC creates a workspace dir, puts identity files in it, and that becomes the agent's home. It does not offer "use this existing directory as the workspace" as a provisioning option (at least not via the UI).

The separation does have one practical benefit: identity files and memory files do not appear in the code repo's git history. But this is a soft preference, not a requirement. Gitignoring the operational files (memory/) and committing the identity files (SOUL.md, AGENTS.md — which are effectively documentation) is a clean model.

**The complexity you are feeling is real and the separation is not load-bearing.** It was created by the tooling, not by necessity.

---

## What would actually change

| Concern | Current (two dirs) | Merged (code repo as workspace) |
|---------|-------------------|---------------------------------|
| Where agent starts | Separate workspace-lead dir | Code repo dir |
| Where identity files live | Workspace-lead dir | Code repo dir |
| Where memory files live | Workspace-lead dir | Code repo dir (gitignored) |
| Access to org-level files | Unrestricted (no sandbox) | Same — unrestricted |
| Code repo git history | Clean | Clean if memory/ gitignored; identity files would appear |
| Workspace-lead dir needed? | Yes | No — can be deleted |
| Retention risk | High (not on GitHub) | Reduced — identity files are now in the code repo on GitHub |

The merged model reduces one of the biggest retention risks identified earlier: the workspace identity files currently live only on the Pi and are not backed up. If the workspace is the code repo, SOUL.md and AGENTS.md are committed and pushed to GitHub as part of the repo.

---

*Compass · Strategic Advisor · IDEA*

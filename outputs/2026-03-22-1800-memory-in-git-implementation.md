> **Question:** Is it sufficient to just document memory-in-git in the design? I want it implemented as well.

---

## The dependency problem

Implementing memory in git properly requires the workspace migration first. Here is why.

Memory files are written by agents to their **workspace directory** — whichever path is set as `workspace` in `openclaw.json`. Right now, that path is the **workspace-lead dirs** (`/home/pi/idea/workspace-lead-{board_id}-*/`), not the code repos.

If I commit the current memory content into the code repos right now, agents will keep writing new memory to the workspace-lead dirs at the next session. The code repos' `memory/` folders become stale immediately — a snapshot, not a live record.

**Memory only stays in git if agents write to the git repo in the first place.**

---

## What is actually in memory right now

Memory lives in two separate places, neither of which is in a GitHub-backed git repo:

| Location | What's there | Git-backed? |
|----------|-------------|-------------|
| `workspace-lead-6bddb9d2-*/memory/` (Axle) | 2026-03-01, 2026-03-19 | Local only |
| `workspace-lead-ac508766-*/memory/` (Pixel) | 2026-03-01, 2026-03-02, 2026-03-19, 2026-03-20 | Local only |
| `workspace-lead-7cc2a1cf-*/memory/` (Beacon) | 2026-03-01, 2026-03-19 | Local only |
| `workspace-lead-d0cfa49e-*/memory/` (Veri) | 2026-03-01, 2026-03-03, 2026-03-19 | Local only |
| `workspace-lead-3f1be9c8-*/memory/` (Marco) | 2026-03-01, 2026-03-19 | Local only |
| `~/.claude/projects/.../memory/` (Compass) | MEMORY.md + project files | Local only |

None of the five code repos (`agent-engine-dev` etc.) have a `memory/` folder at all.

---

## What full implementation requires

Two steps, in order:

### Step 1 — Workspace migration (the prerequisite)

Change each board lead's `workspace` in the openclaw.json Docker volume config from the workspace-lead dir to the code repo:

```
Before: /home/node/workspace/workspace-lead-6bddb9d2-.../
After:  /home/node/workspace/agents/agent-engine-dev/
```

After this change, agents wake up in the code repo. Memory files they write go directly into the code repo. `git add memory/`, `git commit`, `git push` — memory is in GitHub.

### Step 2 — Seed with current memory content

Copy the existing memory files from workspace-lead dirs into the code repos before making the switch, so agents pick up their existing context rather than starting cold.

For Compass: the MEMORY.md and project memory files from `~/.claude/projects/.../memory/` move into `agents/agent-researcher/memory/` in the repo.

---

## What I can do right now vs what needs the migration

| Action | Can do now? | Notes |
|--------|------------|-------|
| Copy existing memory files into code repos | Yes | One-time snapshot; diverges immediately until migration |
| Workspace migration (openclaw.json) | Yes — with your confirmation | Restarts agents' working directory; test that sessions still work |
| Memory stays in git going forward | Only after migration | Agents must write to the repo for commits to capture it |
| Compass memory in git | Yes now | Compass already uses this repo; just move the files in |

---

## Recommended path

Do the workspace migration and memory seeding together as one operation. Separating them creates a window where memory diverges.

**If you want to proceed:** confirm and I will:
1. Copy all current memory content from workspace-lead dirs into code repos
2. Update the `workspace` entries in the openclaw.json Docker volume
3. Restart the OpenClaw gateway to pick up the new config
4. Commit and push the memory seed to each code repo (PR for protected repos)
5. Move Compass's memory into `agents/agent-researcher/memory/` and commit

This is the workspace migration that has been pending since the 2026-03-22 analysis sessions.

---

*Compass · Strategic Advisor · IDEA*

> **Question:** The current engine repo has to migrate into agent-engine-dev and be relocated on the pi to /home/pi/idea/agents. When you will be doing that, can it be done in such a way that (1) the claude session into the new agent-engine-dev has exactly the same memory as the memory into the current agent repo? and (2) the full repo history is kept (past commits)

# Migrating the engine repo to agent-engine-dev

Both constraints are straightforward to satisfy. Here is how each works.

---

## Full git history — `mv` + remote swap

A simple directory move preserves history completely. Git history lives inside the `.git`
folder and travels with the directory:

```bash
mv /home/pi/projects/engine /home/pi/idea/agents/agent-engine-dev
cd /home/pi/idea/agents/agent-engine-dev
git remote set-url origin git@github.com:idea-edu-africa/agent-engine-dev.git
git push -u origin main
```

Every commit, branch, and tag carries over. The new GitHub repo (`idea-edu-africa/agent-engine-dev`)
just needs to exist first — created empty, with no initialisation.

---

## Memory continuity — copy the memory directory

Claude Code stores per-project memory keyed on the project directory path. The path is
encoded by replacing every `/` with `-` (including the leading `/`):

| Location | Encoded path |
|----------|-------------|
| `/home/pi/projects/engine` | `-home-pi-projects-engine` |
| `/home/pi/idea/agents/agent-engine-dev` | `-home-pi-idea-agents-agent-engine-dev` |

The memory directories are therefore:

- **From:** `/home/pi/.claude/projects/-home-pi-projects-engine/memory/`
- **To:** `/home/pi/.claude/projects/-home-pi-idea-agents-agent-engine-dev/memory/`

Copy the memory across **before** opening the first session at the new location:

```bash
mkdir -p /home/pi/.claude/projects/-home-pi-idea-agents-agent-engine-dev/
cp -r /home/pi/.claude/projects/-home-pi-projects-engine/memory/ \
      /home/pi/.claude/projects/-home-pi-idea-agents-agent-engine-dev/memory/
```

The first session opened against the new path will load the copied `MEMORY.md` as if
it had always lived there.

---

## Order of operations

1. Create `idea-edu-africa/agent-engine-dev` on GitHub (empty, no initialisation)
2. Copy Claude memory to new project path (before any session is opened there)
3. `mv /home/pi/projects/engine /home/pi/idea/agents/agent-engine-dev`
4. `git remote set-url origin git@github.com:idea-edu-africa/agent-engine-dev.git`
5. `git push -u origin main`
6. Update `openclaw.json` workspace path: `/home/node/workspace/agents/agent-engine-dev`
7. Restart OpenClaw: `sudo docker restart openclaw-gateway`

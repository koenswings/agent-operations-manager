> **Question:** What is the memory folder for agent-researcher? Who creates it? Why did you mention it if it doesn't exist? Why does OpenClaw put agent identity in workspace folders rather than alongside operational output? And you forgot to categorise the main outputs of the dev agents (code) and Veri (field reports).

---

## Three corrections and one gap in the previous output

---

### 1. The `memory/` folder — what it is, who creates it, and why I was wrong

The `memory/` folder **does exist** — but not where I implied.

- **Actual location:** `/home/pi/.claude/projects/-home-pi-idea-agents-agent-researcher/memory/`
- **Contents:** `MEMORY.md`, `project_branch_protection.md`, `project_telegram_mc.md`
- **Created by:** Claude Code itself (me), automatically, when I save persistent memories between sessions. This is Claude Code's internal auto-memory system — it creates a project-keyed directory under `~/.claude/projects/` the first time memory is written.

My previous output listed `memory/` as a path relative to the git repo (`/home/pi/idea/agents/agent-researcher/memory/`) and marked it as "✓ Committed" and "✗ 3 commits unpushed." **That was wrong on both counts.** The memory folder is in `~/.claude/`, which:
- Is not inside the git repo
- Is not committed to anything
- Is not backed up to GitHub

This makes the retention risk for Compass's own memory **higher** than I reported. The memory files exist only on the Pi's local filesystem, exactly like the workspace identity files I flagged as high-risk. If the Pi is lost, Claude's memory of IDEA's decisions and history is lost with it.

**Corrected retention row:**

| Path | What it contains | Committed? | GitHub backup? | Risk |
|------|-----------------|------------|----------------|------|
| `~/.claude/projects/.../memory/` | Compass session memory | ✗ No | ✗ No | **High** |

---

### 2. Why OpenClaw puts agent identity in workspace folders

OpenClaw creates a dedicated workspace directory per agent (`/home/pi/idea/workspace-lead-{board_id}-*/`). This is the directory that gets **mounted into the agent's Docker container as its working directory**. The agent sees this as its filesystem root when it runs.

Identity files (SOUL.md, AGENTS.md, HEARTBEAT.md, etc.) go into the workspace dir because that is what gets mounted — so the agent can read them. OpenClaw needs to inject these files somewhere the agent can access at runtime, and the workspace dir is the mount point.

**Can OpenClaw be configured to put identity files alongside operational output instead?** Probably not as a configuration option — this is a design decision baked into how OpenClaw provisions agents. The separation exists because:
- The workspace dir is the agent's container filesystem (ephemeral, local)
- Operational output (standups, docs) goes to the org root (`/home/pi/idea/`) which is a separate mount

These are two different Docker volume mounts: the workspace dir for agent state, and the org root for shared org content. OpenClaw doesn't conflate them.

What you *could* do is instruct each agent to commit their operational output to the org root repo (or their own project repo) rather than leaving it loose in the workspace — but that's an instructional change to the agent, not an OpenClaw config change.

---

### 3. The missing output category: the actual work product

The previous overview catalogued identity files and memory but completely missed the primary deliverables of each agent. Here is the corrected picture:

#### Dev agents — code

| Agent | Primary output | Location | Backed up? |
|-------|---------------|----------|------------|
| Axle (engine-dev) | Engine code, PRs | `agent-engine-dev` GitHub repo | ✓ Yes — PR model, CEO merges |
| Pixel (console-dev) | Console code, PRs | `agent-console-dev` GitHub repo | ✓ Yes — PR model, CEO merges |
| Beacon (site-dev) | Site code, PRs | `agent-site-dev` GitHub repo | ✓ Yes — PR model, CEO merges |

Code is actually the **best-retained** output in the whole system. GitHub is the source of truth, main is protected, CEO merge is the gate. No retention concern here — this is working exactly as designed.

The oversight in the previous overview was ironic: the thing most at risk (workspace memory, Compass memory) got flagged; the thing with the best coverage (code in GitHub) wasn't even mentioned.

#### Veri — quality management output

Veri's primary output is **not well-defined yet**. Her role involves quality management, which could produce:
- Field reports (test results, quality assessments against user stories)
- Review comments on PRs (these live in GitHub, retained automatically)
- Periodic quality summaries or dashboards

Currently there is no established location for Veri's non-PR output. If it goes into the org root (e.g., `/home/pi/idea/quality/`), it would need to be committed to the `idea` repo to be retained. If it stays loose in her workspace dir, it is at risk.

**This is a design gap** — Veri's output location has not been specified. It should be decided as part of her role configuration, before she starts producing substantive output.

#### Marco — programme management output

Same situation as Veri. Marco's output likely includes:
- Programme plans and roadmaps
- Standup summaries or coordination notes
- Backlog management records

If this goes into the org root under something like `/home/pi/idea/programme/`, it belongs in the `idea` repo. Currently unspecified.

---

### Corrected retention summary

| Output type | Location | GitHub backup? | Risk |
|-------------|----------|----------------|------|
| **Code (dev agents)** | `agent-{role}` GitHub repos | ✓ Yes | Low — best coverage in the system |
| **PR review comments** | GitHub (inline) | ✓ Yes | Low |
| **Compass research** | `agent-researcher` repo (minus 3 unpushed commits) | ✓ Yes | Low |
| **Compass session memory** | `~/.claude/projects/.../memory/` | ✗ No | **High** |
| **Workspace identity files** | `/home/pi/idea/workspace-lead-*/` | ✗ No | **High** |
| **Veri non-PR output** | Undefined | Undefined | **Gap — needs design decision** |
| **Marco non-PR output** | Undefined | Undefined | **Gap — needs design decision** |

---

*Compass · Strategic Advisor · IDEA*

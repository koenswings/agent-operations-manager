> **Question:** Why are operational agents sandboxed? How? They maintain files in their repo — how can they not push changes or PRs?

## Operational Agent Sandbox — What's Actually True

Short answer: **my previous analysis was wrong.** Operational agents are not sandboxed in the way I described. Let me explain what I found.

---

### What I found in the container

All agents run inside the `openclaw-gateway` Docker container. Checking that environment:

| Capability | Reality |
|-----------|---------|
| `git` installed? | **Yes** — git 2.39.5 |
| Project repos visible? | **Yes** — `/home/node/workspace/agents/agent-engine-dev` etc. all accessible |
| Remote configured on project repos? | **Yes** — `git@github.com:koenswings/agent-engine-dev.git` |
| SSH keys in container? | **No** — no `.ssh` directory exists anywhere |
| GitHub token in environment? | **No** |
| Workspace dirs (workspace-lead-*) — remote configured? | **No** — local-only repos |

---

### The actual isolation mechanism

Operational agents ARE blocked from pushing to GitHub — but not by architecture or interface. They are blocked by **the absence of credentials**.

- `git push` on a project repo would fail: SSH is configured as the remote, but there are no SSH keys in the container.
- `git push` on a workspace dir would fail: no remote is configured at all.
- GitHub API via `curl` would fail: no token.

This is credential-based isolation, not structural isolation.

---

### What this corrects in my previous analysis

The previous output said:

> *"Operational agents via Telegram/OpenClaw have no GitHub capability at all. They operate within OpenClaw's sandbox."*

That was wrong in its reasoning. The correct statement is:

> Operational agents could in principle run git commands and have access to both the workspace dirs and the project repos inside the container. They are prevented from pushing to GitHub because no credentials (SSH key or token) are provisioned in the container — not because of any architectural constraint.

The Telegram interface doesn't sandbox them from git. The missing credentials do.

---

### Does this change the governance design?

**Partially.** The outcome is the same — agents can't push to GitHub — but the guarantee is weaker:

| Claim | Status |
|-------|--------|
| Agents can't push to GitHub | ✓ True — no credentials |
| Agents are architecturally sandboxed | ✗ Wrong — it's credential-based |
| An agent given a GitHub token could push | ✓ Correct |
| Branch protection is the CEO review gate | ✓ Still correct and still necessary |

Branch protection matters more, not less, given this. If credentials were ever accidentally added to the container (e.g., via a git config change or mounted secret), branch protection on the project repos would still block a direct push to `main`. It's the last line of defence.

---

### What's actually true about workspace dirs vs project repos

The distinction still holds, but for different reasons:

| Location | Remote | Why agents don't push |
|----------|--------|----------------------|
| `/home/node/workspace/workspace-lead-*/` | None | No remote configured — structural |
| `/home/node/workspace/agents/agent-*/` | `git@github.com:...` | No SSH key — credential gap |

The workspace dirs are genuinely isolated by structure. The project repos rely on the absence of credentials.

---

### Implications

1. **Don't add SSH keys or tokens to the container** unless you intend agents to have push capability. This is the critical configuration gate.
2. **Branch protection on project repos remains correct** — it's the backstop if credentials ever appear.
3. **The Compass-as-PR-author model is still the right design** — but because it's deliberate policy, not because other agents are technically incapable.
4. **Consider documenting this as an explicit constraint** in TOOLS.md or the openclaw config: "No GitHub credentials are provisioned — agents cannot push to remotes."

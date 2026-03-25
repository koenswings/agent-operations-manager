> **Question:** What are the different options in Claude to restrict the number of approvals I have to give when you are answering a question? And how is it activated?

## Summary

Claude Code has 5 permission modes plus a fine-grained allow/deny rule system. This project currently runs `bypassPermissions` mode — no prompts at all. The allow-list entries in `settings.local.json` are historical and have no effect in that mode.

---

## The 5 Permission Modes

| Mode | What it does |
|---|---|
| `default` | Prompts once per tool type per project directory. File edits prompt every session. Bash commands get a permanent "allow for this dir" option. |
| `acceptEdits` | File edits auto-approved for the session. Bash commands still prompt. |
| `plan` | Read-only — Claude can read and reason but cannot write files or run commands. Nothing executes without you switching mode. |
| `dontAsk` | Denies every tool unless you have an explicit `allow` rule. Opposite of bypassPermissions. |
| `bypassPermissions` | Skips all prompts. Only protected dirs (`.git`, `.claude`, `.vscode`, `.idea`) still require confirmation. |

**Practical range:** `plan` (most restrictive, zero execution) → `default` (one prompt per tool type) → `acceptEdits` (no file edit prompts) → `bypassPermissions` (no prompts at all).

---

## How to Activate

### 1. settings.json (persistent — survives session restarts)

Three file locations, highest to lowest priority:

| File | Scope | Committed to git? |
|---|---|---|
| `.claude/settings.local.json` | This project, you only | No (gitignored) |
| `.claude/settings.json` | This project, whole team | Yes |
| `~/.claude/settings.json` | All your projects | No |

```json
{
  "defaultMode": "bypassPermissions"
}
```

Replace `"bypassPermissions"` with any of the 5 mode names above.

### 2. CLI flag (one session only)

```bash
claude --permission-mode plan
claude --permission-mode acceptEdits
claude --dangerously-skip-permissions    # activates bypassPermissions
```

### 3. Mid-session slash command

Type `/permission-mode` in the Claude terminal to change mode during a running session.

---

## Fine-Grained Allow/Deny Rules

Instead of (or alongside) a global mode, you can allow specific tools and block others:

```json
{
  "permissions": {
    "allow": [
      "Bash(git *)",
      "Bash(npm run *)",
      "Read",
      "WebFetch(domain:github.com)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Read(./.env)"
    ],
    "ask": [
      "Bash(git push *)"
    ]
  }
}
```

**Evaluation order:** `deny` first (always wins), then `ask`, then `allow`. First match wins.

**Rule syntax:**

| Rule | What it matches |
|---|---|
| `Bash(npm run *)` | Any bash command starting with `npm run` |
| `Bash(git commit *)` | Exact prefix match |
| `Read` | All file reads |
| `Read(./.env)` | One specific file |
| `Read(/src/**)` | All files under `/src/` recursively |
| `WebFetch(domain:github.com)` | Web requests to that domain |

---

## This Project's Current Config

`.claude/settings.local.json`:

```json
{
  "defaultMode": "bypassPermissions",
  "permissions": {
    "allow": [ /* ~77 specific rules */ ]
  }
}
```

**`bypassPermissions` is set** — so no approval prompts for anything except protected directories.

**The 77 allow-list entries are not doing anything** in this mode. They were accumulated one by one in earlier sessions before `bypassPermissions` was set — each one was added when a specific command triggered a prompt. Once bypassPermissions was turned on, the list became irrelevant to prompt behaviour. It now serves only as an incidental log of commands that have been run.

If you switched to `default` mode, those 77 rules would kick in and pre-approve everything on that list — reducing prompts significantly without going fully open. But many of the entries are poorly formed (broken `Bash()` patterns from how the allow-list was built interactively) and would not all match correctly.

---

## Recommendations for IDEA Agent Workspaces

| Workspace | Suggested mode | Reason |
|---|---|---|
| `agent-researcher` (Compass) | `bypassPermissions` | Strategic research, no destructive ops, CEO-only access |
| Operational agents (Axle, Pixel, etc.) | `plan` or `default` | They touch code and infra — plan mode enforces CEO approval before anything executes |

`plan` mode is the right default for agents that run autonomously — it means Claude can think and propose but cannot act until you approve the plan.

---

*Compass · Strategic Advisor · IDEA*

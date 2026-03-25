> **Question:** Run in automatic mode always — configure so I do not have to confirm each tool call.

## What Was Done

Set `dangerouslySkipPermissions: true` in `.claude/settings.local.json` for this project.

This replaces the previous long manual allow-list (60+ entries that grew over time) with a single flag that bypasses all confirmation prompts. Takes effect from the next session start.

**File:** `/home/pi/idea/agents/agent-researcher/.claude/settings.local.json`
```json
{
  "dangerouslySkipPermissions": true
}
```

## Scope

Project-level only (`settings.local.json` is local and gitignored). This does not affect:
- Operational agents (engine-dev, console-dev, etc.) — they run via OpenClaw with `DEFAULT_PERMISSION_MODE=plan`
- Any other Claude Code project on the Pi

## To Restart Without Prompts

Close the current terminal and reopen it in VSCode — the tmux session will reattach with the new setting active. Or kill and restart the tmux session:
```bash
tmux kill-session -t claude-agent-researcher
```
Then reopen the VSCode terminal.

> **Task/Question:** Remove the tmux attachment from VSCode. Document SSH session persistence via tmux in the CLAUDE.md chapter. Remove all other tmux references from virtual-company-design.md. Also remove the "What Needs to Happen (in order)" section as it is confusing.

## Changes Made

### `idea/design/virtual-company-design.md` — idea PR #6 (`chore/remove-tmux-vscode`)

1. **Removed "What Needs to Happen (in order)" section** — entire section deleted; it was a setup checklist, all items marked ✅, no longer useful
2. **Removed TOC entry** for "What Needs to Happen (in order)"
3. **Added "Session persistence across SSH disconnects" subsection** in the CLAUDE.md chapter — documents using `tmux new-session` / `tmux attach` after SSHing in to keep `claude` alive across disconnects
4. **Updated backlog entry** from `Set up VS Code / Claude Code / tmux per-project session pattern` → `Set up per-project Claude Code session pattern (SSH; tmux for session persistence)`

### `agent-operations-manager/design/tmux-vscode-setup/agent-terminal-design.md` — pushed to `memory/updates`

- Marked **Superseded** with date and explanation pointing to the new pattern in the design doc
- Retained as historical reference

### `agent-engine-dev/.vscode/settings.json` — agent-engine-dev PR #14 (`chore/remove-tmux-vscode`)

- Removed `terminal.integrated.defaultProfile.linux` and `terminal.integrated.profiles.linux` (tmux profile)
- Kept `typescript.tsdk`

### `agent-researcher/.vscode/settings.json` — local only (repo archived)

- Removed `terminal.integrated.defaultProfile.linux` and `terminal.integrated.profiles.linux`
- Kept `terminal.integrated.scrollback` and `terminal.integrated.inheritEnv`

## Open PRs to Merge

| PR | Repo | Description |
|----|------|-------------|
| #6 | `idea` | Remove VSCode tmux; add SSH session persistence note; remove "What Needs to Happen" |
| #14 | `agent-engine-dev` | Remove tmux terminal profile from .vscode/settings.json |

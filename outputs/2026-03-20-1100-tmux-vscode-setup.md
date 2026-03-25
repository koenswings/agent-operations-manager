> **Question:** In the past you had a document explaining tmux and the setup to auto-connect a vscode terminal so that every agent has its own context / session — please make that doc again (also create a PDF) and expose it in the research folder so that I can refer back to it

## Summary

Recreated the tmux + VSCode terminal isolation reference doc at:

- **Markdown:** `research/tmux-vscode-setup/proposal.md`
- **PDF:** `research/tmux-vscode-setup/proposal.pdf`

The document covers:

1. **How it works** — VSCode's `terminal.integrated.profiles.linux` runs a `tmux attach || tmux new-session` idiom so opening a terminal in any workspace automatically connects to that agent's named session (or creates it).
2. **`~/.tmux.conf`** — explains every setting: `mouse on`, `aggressive-resize`, the `client-attached resize-window -A` hook (fixes the "tiny terminal" problem), 50k scrollback, and the `terminal-overrides` for Shift+copy.
3. **Per-workspace `.vscode/settings.json` template** — copy-paste ready, with optional keys explained.
4. **Session naming table** — current state of all 6 agents (researcher + 5 ops), noting which workspaces still need `.vscode/settings.json` configured.
5. **Setup steps** and **common issues** (the "no current session" error on boot, size glitches, Claude not starting, GPU glitches).

The "no current session" error you saw on startup is normal — it fires when tmux server isn't running yet; the `2>/dev/null` in the command suppresses it and the `||` branch creates a fresh session immediately.

> **Question:** How does a session provide a different context to Claude? Is it only the history of commands given (where is that stored?) or does a session also store any previous output (and where is that stored?)

## Summary

A Claude Code "session" is an in-memory conversation. tmux keeps the `claude` process alive so that memory survives. If `claude` exits, the context is gone — even though artefacts remain on disk, Claude does not reload them on next start.

See full answer in `research/tmux-vscode-setup/agent-terminal-design.md` (Claude Code Session Context section).

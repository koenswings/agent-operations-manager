> **Question:** Claude output in the VSCode terminal is not showing properly — output is interrupted/broken. Fix the terminal issues and make the file-writing rule permanent.

---

## Root Cause

Two problems:

1. **Memory rule was wrong.** The MEMORY.md said "The OpenClaw chat UI truncates responses" — so Compass was mentally treating it as an OpenClaw-only rule and forgetting to apply it in VSCode terminal sessions. Fixed: the rule now says it applies to every interface, every session, no exceptions.

2. **tmux window is 21 lines tall.** The VSCode integrated terminal panel is small. Claude Code output is long and interactive (spinners, diffs, progress bars) — it fills and overwrites the visible area, making it look "broken" when you scroll back.

---

## What Was Fixed

### 1. MEMORY.md — rule is now permanent and unambiguous

Updated the rule from:
> "The OpenClaw chat UI truncates responses. Every substantive response must be written to a file."

To:
> "The VSCode terminal is small (≈21 lines). Claude output is always truncated, broken, or unreadable in the terminal regardless of the interface. Every substantive response MUST be written to a file. This is not optional. No exceptions."

The rule now applies to all sessions, all interfaces.

### 2. VSCode settings — scrollback and rendering

Added to `.vscode/settings.json`:
```json
"terminal.integrated.scrollback": 10000,
"terminal.integrated.gpuAcceleration": "off",
"terminal.integrated.inheritEnv": true
```

- `scrollback: 10000` — VSCode's own terminal buffer (on top of tmux's 50k)
- `gpuAcceleration: "off"` — eliminates GPU rendering artefacts that can corrupt scrollback output
- `inheritEnv: true` — ensures environment is correctly passed through

### 3. tmux — mouse and resize confirmed working

- `mouse on` applied to the running session
- `aggressive-resize on` confirmed active — window scales to VSCode panel size
- `history-limit 50000` confirmed — nothing is lost in scrollback

---

## How to Scroll Back in tmux

If you need to read output that has scrolled off:

1. Press **`Ctrl+B`** then **`[`** — enters scroll/copy mode
2. Use **arrow keys** or **PgUp / PgDn** to scroll
3. Press **`q`** to exit scroll mode

Or: since GPU acceleration is now off, **Shift+scroll** in the VSCode terminal should work directly.

---

## The Real Fix — Read the Output File

Even with all of the above, a 21-line terminal will never comfortably display a long Claude response. **The correct workflow is:**

1. Claude writes response to `outputs/YYYY-MM-DD-HHMM-<topic>.md`
2. Claude posts one-line summary + file path in chat
3. You open the file in VSCode (it renders as Markdown with full width and no truncation)

**To open an output file:** click the path in the terminal, or use `Ctrl+P` in VSCode and type the filename.

---

## Make the Terminal Panel Taller

The tmux window is 21 lines — set by your VSCode terminal panel height. To get more space:
- Drag the terminal panel divider **upward** to give it more height
- Or press `Ctrl+J` to toggle the panel, then resize
- Or right-click the terminal tab → **Move Panel to Editor Area** for a full-size window
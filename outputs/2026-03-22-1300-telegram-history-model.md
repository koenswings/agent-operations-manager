> **Question:** Can an OpenClaw agent be constructed to read its Telegram message history?

---

## Short answer

Not from Telegram's servers — that is a hard Telegram limitation. But within a session, the agent has the current conversation thread as context. Across sessions, it can read history *if it wrote history to its memory files*. There is a practical pattern that gets you most of what you want.

---

## The three layers

### 1. Telegram Bot API — a hard ceiling

Telegram bots receive messages in real time (via webhook). There is no API method to retrieve past messages from a group or DM. Once a message is delivered, bots cannot go back and fetch it.

This is not an OpenClaw limitation. It is a Telegram design decision. Even if you built a fully custom bridge, the ceiling is the same: bots cannot call `getChatHistory`.

---

### 2. OpenClaw — what gets passed to the agent within a session

When a Telegram message arrives, the gateway matches the group ID to an agent via `bindings[]`, then invokes a Claude Code subprocess. What the agent receives is:

- The current message text
- OpenClaw's own stored conversation context for that session thread

The agent does **not** go to Telegram to fetch history. It receives whatever OpenClaw passes from its own internal session store. For an active, ongoing conversation thread, this means the agent has context of prior messages in that same thread — because OpenClaw buffers them, not because Telegram supplies them.

Across separate sessions (different invocations, different days), there is no carry-over from OpenClaw unless it was persisted somewhere. Each new invocation starts from the workspace files.

---

### 3. What is actually readable across sessions — the memory files

The agents already have a memory model designed for exactly this. Each agent reads two things at session start:

- `memory/YYYY-MM-DD.md` — today's (and yesterday's) log
- `MEMORY.md` — durable decisions and state

If an agent is instructed to write key points from each Telegram conversation to its daily memory file, those notes are available at the start of every subsequent session. This is the intended carry-forward mechanism.

**Example instruction in AGENTS.md or SOUL.md:**
> At the end of each session, summarise any decisions or context from the Telegram conversation and append them to `memory/YYYY-MM-DD.md`.

This is not "reading Telegram history" — it is the agent building its own running record of conversations and reading that back. In practice, for the purpose of operational continuity, it achieves the same thing.

---

## What this means for Compass on OpenClaw

If Compass were moved to OpenClaw and accessed via Telegram, the pattern would be:

| Scenario | What happens |
|----------|-------------|
| Ongoing conversation in the same Telegram thread | Compass has full context — OpenClaw passes the thread history to the current session |
| New Telegram message after a gap (new session) | Compass starts from workspace files only — no Telegram history unless it logged it |
| Compass instructed to log key points to memory | Each session begins with those notes — effective continuity across sessions |

The practical conclusion: for a strategic advisor like Compass, the memory file pattern is the right model. Compass should be instructed to close each session by writing a brief note of what was discussed and what was decided. This is more useful than raw Telegram history anyway — it is curated, not a raw transcript.

---

*Compass · Strategic Advisor · IDEA*

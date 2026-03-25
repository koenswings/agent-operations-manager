> **Question:** Will the agents already do some autonomous work before I do the initial conversation?

## Short answer: No meaningful autonomous work will happen before the intro.

The technical bootstrap has already run (that was the one autonomous action). The agents are now idle and will stay that way until something triggers them.

---

## What already happened (autonomously)

When MC provisioned each agent and sent the wakeup message, the agents ran the technical `BOOTSTRAP.md` checklist without you:

- Installed `curl` and `jq`
- Verified API reachability
- Created `memory/` and today's daily file
- Posted a heartbeat → status changed to `online`
- Deleted `BOOTSTRAP.md`

That's done. It was the intended autonomous behaviour. It won't run again.

---

## Why nothing else will happen before the intro

**1. No tasks exist.**
The MC backlog hasn't been migrated yet. When an agent runs its heartbeat loop, it pulls the task queue. An empty queue produces `HEARTBEAT_OK` and nothing else — no decisions, no work.

**2. No heartbeat schedule is configured.**
HEARTBEAT.md says cadence is "controlled by gateway heartbeat config." That config hasn't been set up yet (it's a pending step). Heartbeats aren't firing automatically — the session is idle.

**3. Sessions are idle between messages.**
OpenClaw sessions don't poll or self-invoke. The agents won't wake up unless a message is delivered to them (heartbeat trigger, chat message, task notification).

---

## One thing to be aware of after the intro

The board rule snapshot in HEARTBEAT.md shows:

```
require_approval_for_done: true
```

This means the CEO must approve before a task is moved to `done`. But it does **not** mean agents wait for your sign-off before starting or working on tasks. Once tasks exist and heartbeats are running, agents can create subtasks, assign work, start execution, and comment on tasks — without asking you first.

The approval gate is at the **end** (task closure), not the beginning.

---

## Practical sequence

1. Do (or skip) the personal introduction conversations — your call on timing
2. Set up the heartbeat schedule (pending step)
3. Migrate the backlog into MC boards
4. Once tasks exist and heartbeats fire, agents will start running their board loops

The introduction conversation is for rapport and identity confirmation — it doesn't gate the agents' ability to work. You could skip it entirely and the agents would function, but they'd have the default MC soul (generic lead) rather than the IDEA mission context you've already put in `SOUL.md`.

> **Question:** The budget has been depleted again. Can you check whether agents had been woken up and left traces of activity? Can you lower the heartbeat? Leave the system down.

## Summary

Budget leak confirmed. **Marco (programme manager)** had a cron heartbeat running at 10-minute intervals since March 2, accumulating **2,385 heartbeat runs** before the system was brought down. The other 4 agents have no cron jobs and left no activity traces.

---

## Activity Traces

### What ran on March 18
- Marco's cron heartbeat ran **5 times** between 17:15–17:55 UTC on March 18
- Timestamp evidence: `.pi` directories across all 5 agent workspaces were touched at `17:58` on March 18 — this matches the last heartbeat causing workspace access
- All 5 runs returned `status: ok` (agent ran, made API calls, consumed tokens)
- Run durations: first 9s, last 8s, middle runs ~0.6–0.8s (likely short because MC API was unreachable and pre-flight failed fast)

### All agent MEMORY.md files
All 5 agents show last meaningful activity on **March 2–3**, which matches the known API outage recovery. No substantive work happened after that — boards are empty, all agents were in "waiting" state.

### The .pi directories
Empty directories in each workspace; mtime `Mar 18 17:58`. These appear to be touched by the OpenClaw gateway when accessing the workspace during a heartbeat. Not a sign of agent work — just filesystem access.

---

## Why Only Marco?

Only Marco (board ID `3f1be9c8`) had a cron job configured in OpenClaw's `jobs.json`. The other 4 board leads (Axle, Pixel, Beacon, Veri) have no cron jobs. This likely means Marco's heartbeat was set up during Step 15 configuration but the other agents' heartbeats were either:
- Not yet configured, or
- Configured via the MC UI (not the cron system) and not persisted

**This is actually good news**: only 1 of 5 agents was running heartbeats.

---

## What Changed

**File edited:** `/var/lib/docker/volumes/openclaw_openclaw-data/_data/cron/jobs.json`
**Backup:** `jobs.json.bak-before-interval-change`

| Field | Before | After |
|-------|--------|-------|
| `name` | `Marco heartbeat (10m)` | `Marco heartbeat (60m)` |
| `schedule.everyMs` | `600000` (10m) | `3600000` (60m) |
| `enabled` | `true` | `true` (unchanged) |

The system remains fully down (Docker not running). The change takes effect when you next start OpenClaw. Marco will then run once per hour instead of every 10 minutes — a 6× reduction.

---

## Recommendations

1. **Consider disabling all heartbeats** until tasks are actually assigned to boards. An empty board heartbeat produces no value and just burns tokens. You can re-enable per agent when you're ready to activate them.

2. **Consider 4-hour or 24-hour intervals** for idle boards. Even 60m is high for a board with no tasks. A board check-in makes sense when there's work to coordinate.

3. **Verify the other 4 agents have no hidden cron jobs** — they don't appear in `jobs.json`, but could be configured elsewhere. Current status: confirmed no cron runs for any of the other 4.

4. **Marco's `nextRunAtMs`** in the state still points to the old schedule (10 minutes after last run on March 18). When the system starts, it will likely run Marco immediately (wakeMode: "now") and then shift to the 60m cadence.

---

## System Status

- Docker: **down** (no containers running)
- All agent MEMORY.md: last updated March 2–3 (boards empty, all idle)
- Heartbeat config: **updated to 60m** (will take effect on next system start)

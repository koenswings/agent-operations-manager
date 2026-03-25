> **Question:** Start the implementation. Update the virtual company design doc with this operational flow. Make sure that the memory of all agents are up-to-date with this flow and consistent with it. Define the /standup command.

---

## What was done

### 1. Virtual company design doc — updated ✅
`research/openclaw-initial-config/virtual-company-design.md` updated with Model B operational flow. Key sections rewritten: "A Typical CEO Interaction", "Work Cycle and Automated Behaviour", "Multi-Agent Dialogue", standup and heartbeat redefined. (Completed prior to this session.)

### 2. All 5 agent MEMORY.md files — updated ✅

New `## Operational Model` section added to each agent's MEMORY.md:
- `/home/pi/idea/workspace-lead-6bddb9d2-.../MEMORY.md` — Axle (Engine Dev)
- `/home/pi/idea/workspace-lead-ac508766-.../MEMORY.md` — Pixel (Console Dev)
- `/home/pi/idea/workspace-lead-7cc2a1cf-.../MEMORY.md` — Beacon (Site Dev)
- `/home/pi/idea/workspace-lead-d0cfa49e-.../MEMORY.md` — Veri (Quality Manager)
- `/home/pi/idea/workspace-lead-3f1be9c8-.../MEMORY.md` — Marco (Programme Manager)

The section covers:
- CEO-driven work cycle (nothing moves without a CEO message)
- Auto-review task creation protocol with exact MC API call, all reviewer board IDs
- Hard rule: no task creation from auto-review sessions
- Heartbeat: external event polling only
- Standup: optional, CEO-triggered
- Output type reference (PR / design doc / proposal / report)

Two minor corrections also applied:
- Axle: stale "Next step" reference (task 1bba406c was deleted) corrected to task c74d6a8d
- Pixel: stale heartbeat cadence note updated to new model

### 3. `scripts/check-new-tasks.sh` — written ✅

The pi cron that auto-triggers reviewer agents:

```
/home/pi/idea/scripts/check-new-tasks.sh
```

**Behaviour:**
- Polls all 5 agent boards for tasks tagged `auto-review` with status `inbox`
- Checks `logs/triggered-tasks.log` to prevent double-triggering
- Marks task `in_progress` before recording (prevents race condition)
- Appends task ID to `triggered-tasks.log` (permanent, never deleted)
- Fires isolated OpenClaw gateway session with the task title+description as prompt
- Logs to `logs/check-new-tasks.log`

**Crontab installed** (pi system crontab):
```
*/2 * * * * /home/pi/idea/scripts/check-new-tasks.sh >> /home/pi/idea/logs/check-new-tasks.log 2>&1
```

**Prerequisite:** MC authentication token must be stored at `/home/pi/.mc-token` (single line, the Bearer token value). This file does not yet exist — you need to create it with your MC login token.

### 4. `scripts/standup.sh` — written ✅

The CEO-triggered standup script, activated by `/standup`:

```
/home/pi/idea/scripts/standup.sh
```

**Behaviour:**
1. Runs `standup-seed.sh` to generate/refresh today's standup context file
2. If seed reports no changes (STANDUP_SKIPPED), exits early
3. Fires isolated MC gateway sessions for all 5 agents (Veri first, then dev agents, then Marco)
4. Each agent reads the Context section, appends their `## role` contribution, commits and pushes
5. CEO reads the completed `standups/YYYY-MM-DD.md` and decides whether to adjust any agent's direction

**Standup is NOT wired to Telegram yet.** It fires isolated sessions; agents write to the standup file. Once Telegram group IDs are available, add `delivery.to: "<group_chat_id>"` to the session payload for each agent.

---

## Prerequisite: MC auth token

Both scripts authenticate to MC with a Bearer token. The token must be stored at `/home/pi/.mc-token`:

```bash
# Get your token from an MC login session, then:
echo "your-bearer-token-here" > /home/pi/.mc-token
chmod 600 /home/pi/.mc-token
```

To get the token: log into MC UI in your browser, open DevTools → Network tab → look for the `Authorization: Bearer ...` header on any MC API request.

---

## Wiring `/standup` to Telegram (pending Telegram group IDs)

Once Telegram groups are created, `/standup` can be triggered from Telegram. The approach is to add an OpenClaw command handler in `openclaw.json` that maps `/standup` to running `standup.sh`. Until then, run it directly on the pi:

```bash
/home/pi/idea/scripts/standup.sh
```

---

## What remains

| Item | Status |
|------|--------|
| MC auth token at `/home/pi/.mc-token` | Pending — you need to create this |
| Test `check-new-tasks.sh` manually | Pending — create a test `auto-review` task in MC, wait 2 min |
| Create 5 Telegram groups + update bindings | Pending — group IDs needed |
| Wire `/standup` to Telegram command | Pending — depends on Telegram groups |
| Heartbeat scripts (`heartbeat-ci.sh`, `heartbeat-grants.sh`) | Future — after Telegram groups |
| Regenerate `virtual-company-design.pdf` | Pending |
| MC `restart: unless-stopped` in compose.yml | Pending |

> **Task/Question:** Can you check the status of the install script and ensure it contains all axle's modifications. How can we ensure that this will not happen again, that agents will not touch the files on another repo

## Findings

### run-tests.sh status
- **PR #13 (`chore/ssh-key-design-updates`) is already merged** — `run-tests.sh` is on `main` (commit `f55278f`)
- The Pi just needs `git pull` in `/home/pi/idea/` to make it officially the canonical copy
- Axle confirmed via follow-up: he edited the file directly on the Pi via `sudo sed -i` over SSH while the PR was pending — then Atlas committed the same fix (PR #13), which is now the source of truth

### What actually happened
1. Axle applied `authorized_keys command=` referencing `run-tests.sh` before the file was in `main`
2. Axle then placed the file manually on the Pi via direct SSH write
3. Atlas subsequently committed the correct version via PR #13 (now merged)
4. The file is clean — no drift — but the process was wrong

## Action taken

Opened **idea PR #16** (`chore/infrastructure-discipline`) — adds Infrastructure Discipline section to PROCESS.md with three rules:

### Rule 1: No direct writes to Pi filesystem
Agents must not write files into git-tracked directories on the Pi via SSH, scp, or any remote command. All changes via PR → merge → git pull. Explicit exceptions: `.env`, `authorized_keys`, `~/.openclaw/`.

### Rule 2: Merge before you reference
Live system config (authorized_keys, compose.yaml, cron scripts) must not reference files that aren't in `main` yet. Wrong order: apply config then open PR. Right order: merge PR, Pi pulls, then apply config.

### Rule 3: Agent repo scope
Each agent owns one repo. No commits to, modifications of, or writes into another agent's repo directory — even on the Pi host. The shared `idea` repo requires PRs. Atlas is the only agent authorised to commit to other agents' `memory/updates` branches (via git worktree, per TOOLS.md).

## PR
https://github.com/koenswings/idea/pull/16

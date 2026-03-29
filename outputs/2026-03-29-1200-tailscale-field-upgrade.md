> **Task/Question:** Add a section to the remote management design doc working out a plan to update the current engines in the field with the remote management capability. Including full documentation to tapiwa on how to handle the upgrade and a test of the upgrade here. The current field engines are running the agent-engine-dev code that is tagged with Zimbabwe26012026.

## Research

Inspected Zimbabwe26012026 tag:
- Commit 781c0c2 ("Victoria Falls updates"), tagged 2026-02-07
- OS: Debian arm64, engine at /home/pi/engine/
- Boot: /usr/local/bin/boot.sh (runs at @reboot via cron)
- Daily 5am reboot via crondefs
- No Tailscale installed

## Deliverables

### 1. design/tailscale-remote-management.md — Part 5 added (PR #12)
"Field Upgrade: Retrofitting Existing Engines" section covering:
- What the upgrade adds (binaries, systemd service disabled, auth key, activation script)
- Internet connectivity requirement: Pi needs 4G hotspot at activation time; upgrade itself is offline
- Preparing the USB drive (download Tailscale ARM64 tarball, provision auth key, structure)
- What install.sh does (idempotent, 7 steps)
- Full test procedure on Koen's Pi with pass criteria
- Reference to Tapiwa guide

### 2. scripts/upgrade-tailscale/ (PR #12)
- `install.sh` — offline installer; extracts binaries, installs service (disabled), stores auth key, installs activation script; writes upgrade receipt to /home/pi/upgrade-tailscale.log
- `tailscaled.service` — systemd unit
- `tailscale-debug-activate.sh` — Phase 1 activation script (joins Tailnet, prints IP, waits for Enter, disconnects)
- `README.md` — how to prepare the USB drive

### 3. field/tailscale-debug-upgrade-tapiwa.md (Marco PR #19)
Plain-language guide for Tapiwa:
- What it is and what it doesn't change
- Step 1: install from USB (10 min per school, once only)
- Step 2: activate remote support (phone hotspot + run script + send IP to Koen)
- Troubleshooting section (4 common error cases)
- Which Pis need the upgrade (all shipped before March 2026)

## Test procedure (Koen to run on current Pi)
Documented in design doc Part 5. Requires SSH to Pi host. Steps:
1. Mount USB + run install.sh → verify binaries, service disabled, key stored
2. Run activation script → verify Pi appears in Tailscale admin console
3. SSH in from laptop → verify access
4. Verify Docker/Engine unaffected throughout
5. Press Enter → verify Pi leaves Tailnet (ephemeral key)

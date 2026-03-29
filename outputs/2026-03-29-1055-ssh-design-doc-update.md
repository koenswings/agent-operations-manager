> **Task/Question:** Yes [to: update PR #11 design doc with Tailscale debug mode expansion and Telegram SSH key rotation section]

## Changes made to `design/ssh-key-management.md` (branch `design/ssh-key-management`, PR #11)

### 1. Tailscale debug mode — expanded from one line to a full design sketch

Added a new subsection "Tailscale debug mode (design sketch)" in Part 2 (SSH Access Map), covering:
- How it works (6 steps: install dormant → pre-provision auth key → local activation → `tailscale up` → SSH session → `tailscale down`)
- Design constraints (off by default; no silent remote activation; short-lived auth keys; clean revocation)
- Open design questions (coordinator notification, auth key provisioning, expired key recovery, Console sudo model)
- Status: not yet designed; flagged for pre-deployment design

### 2. Telegram-commanded key rotation — new section in Part 3

Added "Rotating or revoking a key via Telegram command" after "Recommendation for IDEA", covering:
- What Atlas can do autonomously (generate key pair, format authorized_keys line, prepare revocation line, update .env, update keys.md registry)
- What Koen must do on the Pi (append/remove from authorized_keys — one copy-paste command)
- Example Telegram commands
- Scope note: machine-to-machine keys only; Koen's personal laptop key is out of scope

## Commit
`e2e50bf` — pushed to `origin/design/ssh-key-management`

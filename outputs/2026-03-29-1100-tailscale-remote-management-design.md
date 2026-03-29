> **Task/Question:** OK. Please document the use of an ephemeral key then. And start writing a design doc for remote management using Tailscale working out the sketch. Please refer to this doc from the key management design doc

## New doc: `design/tailscale-remote-management.md` (PR #12)

Full design for Tailscale field debug mode. Sections:

1. **Background** — why remote access is needed without compromising offline-first
2. **Design principles** — off by default; local activation only; ephemeral sessions; auditable
3. **Tailscale plan and cost** — free Personal tier; £0 at current scale; ephemeral keys keep device count near zero
4. **Ephemeral auth keys** — regular vs ephemeral comparison; reusable ephemeral key design; storage at `/etc/tailscale/debug-authkey` (600, root-owned)
5. **Network and access control** — ACL tags (`tag:idea-ops`, `tag:school-pi`); SSH key with `from=` restriction (full shell, bounded by Tailscale ACL)
6. **Activation mechanisms** — Phase 1: USB script (immediate, no Engine changes); Phase 2: Console UI toggle with Engine API + narrow sudoers
7. **Session flow** — end-to-end coordinator ↔ IDEA staff ↔ Pi diagram
8. **Key lifecycle** — imaging → dormant → session → rotation; recommendation for "no expiry" on school Pi auth keys
9. **Open questions** — coordinator IP relay method; per-Pi vs per-batch keys; auto-timeout; MagicDNS stability; multi-Pi schools
10. **Implementation plan** — Phase 1 checklist (no cross-agent tasks); Phase 2 tasks (Axle + Pixel, deferred)

## Updated: `design/ssh-key-management.md` (PR #11)

- Tailscale debug mode section rewritten: replaced sketch with concise summary + code snippets + reference to new doc
- `authorized_keys` summary map updated: removed "not yet designed"; shows ephemeral + ACL approach
- Key registry example updated: added `id_ed25519_field_debug` and `tailscale-school-pi-authkey` rows

## PR references
- PR #11 (`design/ssh-key-management`) — updated, references PR #12
- PR #12 (`design/tailscale-remote-management`) — new, commit `89ceb6c`

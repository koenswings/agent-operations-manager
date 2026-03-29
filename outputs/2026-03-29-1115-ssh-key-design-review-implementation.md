> **Task/Question:** OK. I merged PR 11. But both Axle and I made comments - take them into consideration. Update the design accordingly and proceed with the implementation.

## Review comments addressed (4 items from Axle, confirmed by Koen)

1. **Wrapper script pattern** (Comment 1): `command=` in `authorized_keys` now points to `scripts/run-tests.sh` instead of inline command chain. Documented as the required pattern for all machine keys. Wrapper handles `git pull` sequencing cleanly.

2. **from= subnet not specific IP** (Comment 2): Changed from specific container IP to `172.20.0.0/24`. Added note: hardcoding a container IP will silently break across Docker restarts.

3. **Kit key must be separate** (Comment 3): Made explicit — Kit uses same SSH mechanism but its own key (`id_ed25519_ocl_app_tests`) with its own `command=` pointing to `run-app-tests.sh`. Independent revocation and audit trail.

4. **Key comment fix** (Comment 4): Noted in the access map entry — existing `openclaw-axle@idea` comment to be corrected to `openclaw-container→pi-host-tests [2026-03-28]` when restriction is applied.

## Files created/updated

- **`design/ssh-key-management.md`** — updated with all four review changes
- **`platform/keys.md`** — new key registry; one entry per key; covers active, planned, and revoked; rotation policy included
- **`scripts/run-tests.sh`** — new wrapper script; target of `command=` restriction; handles git pull + pnpm install + build + test

All in **idea PR #13** (`chore/ssh-key-design-updates`).

## Cross-agent task created

**Axle board task `ebfa743b`**: `[From Atlas] Implementation: apply command= restriction to Pi host authorized_keys`

Specifies the three remaining Pi-side steps:
1. Edit `authorized_keys` with correct `command=` + `from=172.20.0.0/24` + `restrict` + updated key comment
2. Review `scripts/run-tests.sh` from PR #13
3. Add `TEST_SSH_KEY` to `.env` and verify end-to-end

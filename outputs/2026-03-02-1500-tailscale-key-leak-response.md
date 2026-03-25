> **Question:** GitGuardian detected a Tailscale Pre-Authentication Key exposed in koenswings/app-openclaw (pushed Feb 21 2026).

## Immediate Security Response

**Exposure window:** Feb 21 to now (~9 days). Act immediately.

---

## Step 1 — Revoke the key NOW (< 5 minutes)

1. Go to [https://login.tailscale.com/admin/settings/keys](https://login.tailscale.com/admin/settings/keys)
2. Find the exposed pre-auth key (look for one created around Feb 21)
3. **Delete / revoke it immediately**

Once revoked, the key cannot be used to join new devices to your tailnet — even if someone copied it.

---

## Step 2 — Check for unauthorized devices

1. Go to [https://login.tailscale.com/admin/machines](https://login.tailscale.com/admin/machines)
2. Review every device — look for anything you don't recognise, especially machines joined on or after Feb 21
3. **Remove any unknown devices**

---

## Step 3 — Remove the secret from the repository

The key is in `koenswings/app-openclaw`. Remove it from the current codebase:

```bash
# On the machine with the repo checked out
# Find the file containing the key
grep -r "tskey-auth" .

# Replace with a placeholder or environment variable
# Then commit and push
```

If the key is in a `compose.yaml` or `.env` file, move it to an environment variable or secret manager (e.g. a `.env` file that is gitignored, or a GitHub Actions secret).

---

## Step 4 — Clean git history (optional but recommended)

The key is already revoked so history rewriting is lower priority, but for hygiene:

```bash
# Using git-filter-repo (preferred over BFG)
pip install git-filter-repo
git filter-repo --replace-text <(echo 'tskey-auth-XXXX==>REDACTED')
git push --force
```

**Only do this if the repo is not widely forked** — force-push rewrites history for all collaborators.

Alternatively: leave the revoked key in history (it's harmless once revoked) and just ensure the current code doesn't reference it.

---

## Step 5 — Generate a new pre-auth key if needed

1. Go to [https://login.tailscale.com/admin/settings/keys](https://login.tailscale.com/admin/settings/keys)
2. Generate a new key with the minimum necessary permissions (ephemeral if for CI/containers)
3. Store it in a **GitHub secret** (`Settings > Secrets and variables > Actions`) or a `.env` file that is in `.gitignore`
4. Reference it as `${{ secrets.TAILSCALE_AUTHKEY }}` in compose or CI config

---

## Risk Assessment

| Risk | Likelihood | Notes |
|------|-----------|-------|
| Unauthorised device joined tailnet | Low-moderate | Pre-auth keys require knowing your tailnet; check machine list |
| Key used in automated attack | Low | Tailscale keys are targeted, not mass-scanned like AWS keys |
| Ongoing exposure | **Resolved once revoked** | Revocation is immediate and complete |

---

## Summary

Priority order:
1. **Revoke the key** in Tailscale admin (do this now)
2. **Check machine list** for unknown devices
3. **Remove key from repo code** and replace with a secret reference
4. Optionally clean git history

The 9-day exposure is a concern but pre-auth keys only allow *joining* your tailnet — they don't grant access to existing machines or data. Once revoked, risk is fully contained.

> **Question:** For branch-protected repos, do identity files have to be updated via a PR, or can GitHub be configured so identity files can be pushed directly while code changes still require a PR?

---

## Short answer

GitHub branch protection works at the **branch level**, not the file level. There is no native mechanism to say "require PR for `*.ts` but allow direct push for `SOUL.md`." The choice is: require PRs for everything, or allow direct push for everything (for a given actor).

There is a way to get close to what you want, but with an important caveat specific to this setup.

---

## What GitHub actually offers

### Classic branch protection (what the repos currently use)

Current state of `agent-engine-dev`:
- PR required before merging to main ✓
- Required approvals: **0** (PR must be opened but nobody needs to approve it)
- `enforce_admins: true` (even the repo owner must go through a PR)
- No bypass actors configured
- No rulesets

Classic branch protection applies to the whole branch. You can configure:
- **Bypass actors** — specific GitHub users, teams, or apps that are exempt from the PR requirement and can push directly to main. Anyone not on the bypass list still needs a PR.

This is the closest thing to what you want: add the CEO's GitHub account to the bypass list, and the CEO can push directly. Agents, if they had separate credentials, could not.

### GitHub Rulesets (the newer system)

GitHub Rulesets allow more granular rules, including **file path restrictions** — but these are about *preventing* pushes to certain paths, not about *different PR requirements per path*. You still cannot say "this path requires a PR, that path doesn't" within a single branch.

### CODEOWNERS

Lets you require specific reviewers for specific file paths. Does not affect whether a PR is required at all — only who must approve it. Not relevant here.

---

## The complication specific to this setup

On this Pi, all agents run using the **same git credentials** — your SSH key (or your PAT). From GitHub's perspective, a push from Compass and a push from Axle look identical: same user, same repo.

If you add `koenswings` as a bypass actor on the branch protection, you effectively allow **any process running on this Pi** to push directly to main — because they all authenticate as you. There is no technical enforcement separating "Compass pushing SOUL.md" from "Axle pushing code."

The distinction between Compass doing identity file maintenance and Axle doing code work is **instructional** (they are told different things in their role files), not credential-enforced.

---

## The practical options

### Option A — Keep PRs for everything, make them zero-friction

With `required_approving_review_count: 0`, a PR needs no approvals. Compass opens a PR with identity file changes and can merge it immediately:

```bash
git checkout -b update-soul-axle
git add SOUL.md AGENTS.md
git commit -m "Update Axle identity files"
git push -u origin update-soul-axle
gh pr create --fill
gh pr merge --merge
```

Or in one step: `gh pr create --fill && gh pr merge --merge`

This adds about 10 seconds and creates a clean audit trail. The code repo history shows who changed identity files and when. Every change is intentional and logged.

**This is the recommended option for this setup.** The overhead is minimal, and the audit trail is valuable.

### Option B — Remove `enforce_admins`, accept the trust model

Remove `enforce_admins: true`. This allows the repo owner (you) to push directly to main. Since all agents use your credentials, this is not really "allowing the CEO and blocking agents" — it is "trusting that the instructional layer is sufficient."

You would be relying on the fact that agents are instructed not to push to main directly. If an agent misbehaves or misunderstands its instructions, it could push directly. Given that plan mode requires your approval before agents execute, this risk is low but real.

To remove enforce_admins via CLI:
```bash
gh api repos/koenswings/agent-engine-dev/branches/main/protection/enforce_admins \
  -X DELETE
```

### Option C — Per-agent deploy keys (future, not worth it now)

Give each agent a unique deploy key with limited scope. GitHub deploy keys can be read-only or read-write, but still cannot enforce file-path-level restrictions. You'd need a server-side git hook or a CI check to enforce path restrictions per key. This is significantly more complex and not worth pursuing.

---

## Recommendation

**Use Option A.** Keep PRs required for everything. For identity file updates, Compass opens and self-merges a PR in two commands. This costs almost nothing, and you get:

- A clean audit trail of every identity file change
- Clear separation between "code PR opened by agent" and "identity file PR opened by Compass"
- No weakening of the branch protection that currently guards code changes

The PR requirement is not the friction point. The friction in the current setup is that identity files are not in the code repos at all. Once they are, a self-merging PR for an identity file update is a minor, deliberate action — not a burden.

---

*Compass · Strategic Advisor · IDEA*

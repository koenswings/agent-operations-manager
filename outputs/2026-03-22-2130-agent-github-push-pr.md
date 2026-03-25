> **Question:** How do agents autonomously create PRs on GitHub? How do we correctly instruct all agents to do it?

---

## What is and isn't available in the container

| Tool | Available | Notes |
|------|-----------|-------|
| `git` | ✅ | Full git client |
| `gh` CLI | ❌ | Not installed |
| HTTPS to GitHub | ✅ | Outbound HTTPS works |
| SSH to GitHub | ❌ | No keys in container |
| `curl` | ✅ | Available |

Current remote URLs are SSH (`git@github.com:...`) — these won't work. HTTPS with a token will.

---

## The solution: GITHUB_TOKEN in `.env` + HTTPS remote + GitHub API via curl

### One-time setup (CEO does this once)

1. Generate a GitHub PAT at `https://github.com/settings/tokens`
   - Classic token, `repo` scope
   - Expiry: no expiry (or 1 year — your call)

2. Add it to each agent's `.env`:
   ```
   GITHUB_TOKEN=ghp_xxxxxxxxxxxxxxxxxxxx
   ```
   `.env` is gitignored — never committed.

3. That's it. Agents handle everything else from here.

### What agents do (fully autonomous)

**Push a branch:**
```bash
source .env
git remote set-url origin https://koenswings:${GITHUB_TOKEN}@github.com/koenswings/REPO_NAME.git
git push origin BRANCH_NAME
git remote set-url origin https://github.com/koenswings/REPO_NAME.git
```
The token is set temporarily for the push, then removed from the URL so it's never stored in git config.

**Open a PR:**
```bash
source .env
curl -s -X POST "https://api.github.com/repos/koenswings/REPO_NAME/pulls" \
  -H "Authorization: token ${GITHUB_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{
    \"title\": \"PR TITLE\",
    \"head\": \"BRANCH_NAME\",
    \"base\": \"main\",
    \"body\": \"PR body text\"
  }" | grep '"html_url"'
```

This uses only `git` and `curl`, both available in the container.

---

## What changes in the identity files

**TOOLS.md** — each agent gets a `## GitHub Push & PR` section with the exact commands above (repo-specific).

**SOUL.md** — the PR output type description now says agents push and open PRs themselves. No "tell CEO" language.

Both changes go into each agent repo via PR on branch `fix/autonomous-github-workflow`.

---

## One PAT for all agents, or one per agent?

One PAT is fine for this setup. All agents authenticate as `koenswings`. If you want per-agent accountability in GitHub's audit log, you'd need separate GitHub accounts — not worth it at this scale.

---

*Compass · Strategic Advisor · IDEA*

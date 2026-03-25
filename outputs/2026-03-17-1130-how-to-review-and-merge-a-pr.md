> **Question:** How does a PR get reviewed, approved, and merged in this project?

---

# How to Review and Merge a Pull Request

This guide covers the full PR workflow for the IDEA project. All code changes made by agents go through a PR — the CEO (you) reviews and merges. This is a deliberate safety gate: no agent can change production code without your approval.

---

## The Workflow at a Glance

```
Agent writes code
      ↓
Agent opens PR on a feature branch
      ↓
CEO reviews the diff on GitHub
      ↓
CEO merges to main
      ↓
(Optional) CEO pulls changes to the Pi
```

---

## Step 1 — Find the PR

When an agent opens a PR, it will give you a GitHub URL like:
```
https://github.com/koenswings/app-openclaw/pull/1
```

Open that URL in your browser. You will see:
- **Title** — what the change does
- **Description** — why it was made and what to watch for
- **Files changed** tab — the exact code diff

---

## Step 2 — Review the Diff

Click the **"Files changed"** tab. You will see lines in red (removed) and green (added).

What to check:
- Does the change match what you asked for?
- Are any secrets, passwords, or API keys visible in green lines? (There should never be — raise with the agent if so)
- Does anything look unrelated to the stated purpose?

For this project you don't need to understand every line of code. The key question is: **does this do what was described, and nothing else?**

---

## Step 3 — Approve (optional for this repo)

This repo is configured to allow merging without a formal approval (0 required reviewers). You can still leave a review if you want:

1. On the **Files changed** tab, click **"Review changes"** (top right)
2. Select **"Approve"**
3. Click **"Submit review"**

This is optional — you can go straight to merging.

---

## Step 4 — Merge the PR

Go back to the **Conversation** tab (the main PR page).

Scroll to the bottom. You will see a green **"Merge pull request"** button.

**Three merge options** — click the small arrow next to the button to choose:

| Option | What it does | When to use |
|--------|-------------|-------------|
| **Create a merge commit** | Keeps all commits + adds a merge commit | Default — use this for most changes |
| **Squash and merge** | Collapses all commits into one | Use when the branch has many messy "wip" commits |
| **Rebase and merge** | Replays commits directly onto main, no merge commit | Use when you want a clean linear history |

For this project: **"Create a merge commit"** is the default and works fine for everything.

1. Click **"Merge pull request"**
2. Click **"Confirm merge"**
3. GitHub will offer to delete the branch — click **"Delete branch"** to keep things tidy

The PR is now merged. ✓

---

## Step 5 — Pull the changes to the Pi (if needed)

Merging on GitHub updates the remote repo. If you need the changes on the Pi immediately (e.g. to restart a Docker stack), pull them:

```bash
cd /home/pi/openclaw        # or whichever repo was changed
git checkout main
git pull
```

Then restart the relevant stack if needed:
```bash
cd /home/pi/openclaw
docker compose up -d
```

---

## Using the CLI instead of the browser

If you prefer to stay in the terminal, `gh` (GitHub CLI) can do everything:

```bash
# View the PR
gh pr view 1

# See the diff
gh pr diff 1

# Merge it (merge commit, then delete branch)
gh pr merge 1 --merge --delete-branch
```

---

## Repo Locations and Their PRs

| Repo | Local path | GitHub |
|------|-----------|--------|
| `app-openclaw` | `/home/pi/openclaw` | github.com/koenswings/app-openclaw |
| `idea` (org root) | `/home/pi/idea` | github.com/idea-edu-africa/idea |
| `agent-researcher` | `/home/pi/idea/agents/agent-researcher` | github.com/idea-edu-africa/agent-researcher |
| `agent-engine-dev` | `/home/pi/idea/agents/agent-engine-dev` | github.com/idea-edu-africa/agent-engine-dev |
| `agent-console-dev` | `/home/pi/idea/agents/agent-console-dev` | github.com/idea-edu-africa/agent-console-dev |
| `agent-site-dev` | `/home/pi/idea/agents/agent-site-dev` | github.com/idea-edu-africa/agent-site-dev |
| `agent-quality-manager` | `/home/pi/idea/agents/agent-quality-manager` | github.com/idea-edu-africa/agent-quality-manager |
| `agent-programme-manager` | `/home/pi/idea/agents/agent-programme-manager` | github.com/idea-edu-africa/agent-programme-manager |

---

## What Happens if a PR Has Problems

If you see something wrong in the diff:

1. **Leave a comment** on the PR (on GitHub, click the `+` next to a line to comment inline, or use the comment box at the bottom)
2. **Close the PR without merging** — scroll to the bottom, click "Close pull request"
3. Go back to the agent and explain what needs to change

The agent will fix it and open a new PR (or push to the same branch, which updates the existing PR automatically).

---

## Branch Protection

The `main` branch on `app-openclaw` is protected:
- Direct pushes are blocked — changes must go through a PR
- No approvals are required (you are the sole reviewer)
- Admins are also subject to this rule (no bypassing)

This is intentional. Even when working alone, PRs create an audit trail and a moment to review before code reaches production.

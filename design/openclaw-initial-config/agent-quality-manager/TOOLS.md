# TOOLS.md — Quality Manager

## Environment

- **QM workspace:** `/home/node/workspace/agents/agent-quality-manager`
- **Org root:** `/home/node/workspace/` (CONTEXT.md, BACKLOG.md, proposals/, standups/)
- **Read access to all agent repos:** `/home/node/workspace/agents/`

## GitHub Access

Use `gh` CLI to list and review open PRs:
```
gh pr list --repo koenswings/agent-engine-dev
gh pr list --repo koenswings/agent-console-dev
gh pr list --repo koenswings/agent-site-dev
gh pr view <number> --repo koenswings/<repo>
gh pr diff <number> --repo koenswings/<repo>
```

Leave review comments:
```
gh pr comment <number> --repo koenswings/<repo> --body "..."
```

## Notes

_(Repos currently under `koenswings`; will transfer to the org once the name is confirmed. Update repo paths at that point.)_

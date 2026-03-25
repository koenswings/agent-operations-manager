# TOOLS.md — Site Developer

## Environment

- **Website repo:** `/home/node/workspace/agents/agent-site-dev`
- **Org root:** `/home/node/workspace/` (CONTEXT.md, BACKLOG.md, proposals/, etc.)
- **GitHub Pages:** deploys automatically from `main` via GitHub Actions

## Content Source

- Website content drafts from Programme Manager: `content-drafts/` in this repo
- Programme Manager opens PRs with draft files; pick up, implement, open build PR

## Deployment

- Push to `main` (via merged PR) → GitHub Actions builds → GitHub Pages updates
- Check Actions tab on GitHub to verify deploy status

## Notes

_(Add Astro/Hugo version details, plugin notes, or deployment quirks here as discovered.)_

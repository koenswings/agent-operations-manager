# TOOLS.md — Atlas, COO & Quality Manager

## Environment

- **My repo:** `/home/node/workspace/agents/agent-operations-manager/` (= `/home/pi/idea/agents/agent-operations-manager/` on host)
- **Org root:** `/home/node/workspace/` (CONTEXT.md, BACKLOG.md, ROLES.md, skills/, agents/, etc.)
- **Projects root (host):** `/home/pi/idea/`
- **Shared skills:** `/home/node/workspace/skills/`
- **All agent repos:** `/home/node/workspace/agents/agent-*/`

## MC API

- `BASE_URL=http://172.18.0.1:8000`
- `AUTH_TOKEN` — load from `.env` in this directory (gitignored, never committed)
- No dedicated MC board for Atlas (strategic/ops role — work tracked in git, not MC tasks)
- Use AUTH_TOKEN to read other agents' boards for cross-agent coordination and quality review

See the **mc-api** shared skill for OpenAPI refresh, discovery policy, and usage examples:
`/home/node/workspace/skills/mc-api/SKILL.md`

## Telegram

- Bot token: read from `/root/.openclaw/openclaw.json` → `channels.telegram.botToken`
- To send a file directly (e.g. a PDF): `curl -F document=@/path/to/file.pdf "https://api.telegram.org/bot<TOKEN>/sendDocument?chat_id=<CHAT_ID>"`
- My group chat ID: `-5105695997`

## GitHub Push & PR

`gh` is not available in the sandbox. Use `git` + `curl` with `GITHUB_TOKEN` from `.env`.

### Push a branch
```bash
source .env
git remote set-url origin https://koenswings:${GITHUB_TOKEN}@github.com/koenswings/agent-operations-manager.git
git push origin BRANCH_NAME
git remote set-url origin https://github.com/koenswings/agent-operations-manager.git
```

### Open a PR
```bash
source .env
curl -s -X POST "https://api.github.com/repos/koenswings/agent-operations-manager/pulls" \
  -H "Authorization: token ${GITHUB_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{
    \"title\": \"PR TITLE\",
    \"head\": \"BRANCH_NAME\",
    \"base\": \"main\",
    \"body\": \"PR description\"
  }" | python3 -c "import sys,json; print(json.load(sys.stdin).get('html_url','error'))"
```

`GITHUB_TOKEN` must be present in `.env` (gitignored, never committed).

## Notes

_(Add setup quirks, path changes, or environment-specific observations here as discovered.)_

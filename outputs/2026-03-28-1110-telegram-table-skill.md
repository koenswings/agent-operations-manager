> **Task/Question:** Is it worth documenting the table-as-image approach as a skill? How do you ensure all agents will do it? Go ahead.

## What Was Done

### 1. `telegram-table` skill created
- Location: `/home/node/workspace/skills/telegram-table/`
- `SKILL.md`: frontmatter description triggers the skill whenever an agent needs to display a table in Telegram; body contains usage instructions and column-width guidance
- `scripts/render_table.py`: full CLI script using ImageMagick; auto-sizes columns, alternating row shading, dark header; accepts `--headers`, `--rows`, `--col-widths`, `--font-size`; tested and working

### 2. PROCESS.md updated
- New "Communication Standards" section at the end
- States the rule (no markdown/ASCII tables in Telegram) and points to the skill

### 3. All 5 agents' MEMORY.md files updated
- Each now contains a "Communication Standards" section with the rule and skill path
- MEMORY.md is read at every session start — strongest enforcement lever

## Enforcement layers (in effect)
- **Skill description** — agents scan available_skills; description says "use this whenever you need a table in Telegram"
- **MEMORY.md** — hard rule read every session, applies even without explicit skill lookup
- **PROCESS.md** — org-level standard; authoritative source

## Commits
- idea `chore/post-migration-updates`: skill + PROCESS.md (10th commit on PR #9)
- agent-engine-dev `memory/updates`: MEMORY.md update
- agent-console-dev `memory/updates`: MEMORY.md update
- agent-site-dev `memory/updates`: MEMORY.md update
- agent-programme-manager `memory/updates`: MEMORY.md update (cherry-picked from wrong branch)
- agent-operations-manager `memory/updates`: this output file

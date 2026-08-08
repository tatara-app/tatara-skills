# Tatara Agent Skills

[![skills.sh](https://skills.sh/b/tatara-app/tatara-skills)](https://skills.sh/tatara-app/tatara-skills)

Agent Skills that teach AI agents to work with [Tatara](https://tatara.app) —
the governed document store your company's humans and agents share — through
the Tatara MCP server. Skills follow the open
[Agent Skills](https://agentskills.io) standard and work with Claude Code,
Cursor, Codex, GitHub Copilot, Gemini CLI, and 70+ other agents.

Tatara's write gates enforce canonical document formats. These skills
front-load the grammar so your agent's first write passes the gate instead of
bouncing off it.

## Install

All skills, into every agent detected on your machine:

```bash
npx skills add tatara-app/tatara-skills
```

One skill, or specific agents:

```bash
npx skills add tatara-app/tatara-skills --skill tatara-kanban
npx skills add tatara-app/tatara-skills -a claude-code -a cursor
```

As a Claude Code plugin:

```bash
claude plugin marketplace add tatara-app/tatara-skills
claude plugin install tatara@tatara
```

On claude.ai (web): download this repo, zip a folder under `skills/`, and
upload it in **Settings → Capabilities → Skills**.

## Skills

| Skill | Teaches your agent to |
| --- | --- |
| `tatara` | Work the Tatara MCP end-to-end: search before writing, resolve folders, tag from the taxonomy, pick a render format, recover from gate rejections. |
| `tatara-kanban` | Author kanban boards: `## Column` headings + `- [ ]` cards, canonical grammar, minimal-diff edits. |
| `tatara-tracker` | Author task trackers: sections, task items, and signifier tokens (tags, priority, 📅 due, ✅ done). |
| `tatara-table` | Author typed data tables: one pipe table + typed columns (`render_config`), formula columns. |
| `tatara-base` | Author bases (saved views): the single `base`-fenced YAML block — filters, properties, views. |
| `tatara-mindmap` | Author mind maps: `## Heading` branches off the document title, nested `- ` topics, subtree edits. |

## License

MIT

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
| `tatara-dashboard` | Author dashboards (workspace homepages): rows of live stat / ratio / list / table / cards / bar blocks plus links, prose, and callouts. |
| `tatara-html` | Author HTML documents: reports, decks, and interactive prototypes whose scripts run in a sandbox (Motion, React, `htm` by name), the headless-browser check every write returns and how to fix from it, and images from workspace assets, inline SVG, or `data:` URIs. |
| `tatara-sheets` | Build and change workbooks: tables with typed columns, computed columns, and named cells with live formulas; read, filter, and calculate without reading every row; resend safely with `op_id`; recover from each refusal. |
| `tatara-slides` | Author slide decks: one `<section id>` per slide on a 1920×1080 canvas, read by outline and line range, edit with `base_hash` so concurrent writers never overwrite each other, and fix the slides the render check says do not fit. |
| `tatara-data` | Build and change data pages: stats, charts, tables, and sentences bound to a workbook's named cells and columns, with every figure put in the workbook first, display keys such as `decimals`, a result that shows what each block now displays and where its figures come from, and recovery from each refusal. |
| `tatara-whiteboard` | Build and change whiteboards: notes in sections with named relations between them, a vocabulary each whiteboard declares, reads from a small outline, a walk along one relation to its end, batches that several agents can write at once, and recovery from each refusal. |

## License

MIT

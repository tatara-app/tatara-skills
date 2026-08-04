---
name: tatara-tracker
description: Authors and edits Tatara task tracker documents (render_format tracker) - checklists with optional `## Section` headings and `- [ ]` task items carrying signifier tokens for tags (#tag), priority, due date (📅 YYYY-MM-DD), and completion date (✅ YYYY-MM-DD). Use when creating or updating a task list, todo list, or tracker in Tatara, or when a write is rejected with tracker format errors. Covers loose tasks before the first section, date validity, duplicate-signifier rules, detail lines, and disallowed prose.
license: MIT
compatibility: Requires the Tatara MCP server connected
metadata:
  author: tatara-app
---

# Tatara task trackers

## Format contract

- Trackers are markdown documents with `render_format: "tracker"`. Create and
  update them with `Tatara:write_document`, passing `render_format: "tracker"`
  on create.
- This skill is the body grammar only. The write workflow — search first,
  resolve the folder and tags with `Tatara:get_taxonomy`, and the fields a
  create requires (`title`, `folder`, `type`, `description`, `tags`) — is
  the core `tatara` skill; follow it alongside this one.
- The tracker format takes no configuration in v1 — grouping and filters are
  chosen in the UI toolbar. Omit `render_config`; a non-empty config is
  rejected.
- The write gate parses the body before saving. A body that does not parse as
  tracker is rejected with structured errors (codes below, each with a line
  number and fix instruction) plus a canonical example. A body that parses but
  is not byte-canonical is rejected with the exact corrected bytes in
  `canonical_body` — resend that, never retry blind.
- Checked `[x]` means done. Grouping, filtering, and progress in the Tatara UI
  are view state only — they never change the markdown.

## Grammar

- A tracker is a task list, optionally grouped: `## Section` headings open
  sections; top-level `- [ ]` (open) / `- [x]` (done) list items are tasks.
- Loose tasks are allowed only before the first `## Section` heading — they
  form one implicit ungrouped section at the top. After any heading, every
  task belongs to a section.
- Empty sections are allowed: a heading with no tasks.
- Lines indented under a task (by two spaces, as in the canonical example)
  are detail text, preserved verbatim. Indented `- [ ]` items under a task
  are subtasks (kept as detail, not tracked separately).
- Nothing else. No prose before, between, or after sections, no `#` H1
  headings, no code fences, tables, or blockquotes.

Emit canonical bytes exactly: `-` bullets, lowercase `[x]`, one blank line
after a heading that has tasks, one blank line between sections, tasks on
adjacent lines. Extracted signifiers sit at the end of the title line in
fixed order — priority, then 📅 due, then ✅ done — separated by single
spaces; tags stay wherever they appear in the title. Write bare emoji
without the U+FE0F variation selector. The document ends immediately after
its last content line with a single newline — a trailing empty section ends
right after its `## Heading` line, with no trailing blank line.

Near-miss markdown (`*`/`+` bullets, `1.`/`1)` ordered items, bullets with
no checkbox, `###`–`######` headings, uppercase `[X]`, extra blank lines,
signifiers written mid-title) parses but is rejected as non-canonical, with
the corrected `canonical_body` returned. Bullets without a checkbox parse as
unchecked tasks.

## Signifiers

Inline tokens on a task's title line:

| Token | Meaning | Value rules |
| --- | --- | --- |
| `#tag` | Tag | Letters, digits, `/`, `_`, `-`; at least one non-digit character. Part of the title text — stays where written, may sit mid-title. Indexed for grouping and filtering. |
| 🔺 | Priority: highest | At most one priority glyph per task. |
| ⏫ | Priority: high | At most one priority glyph per task. |
| 🔼 | Priority: medium | At most one priority glyph per task. |
| 🔽 | Priority: low | At most one priority glyph per task. |
| ⏬ | Priority: lowest | At most one priority glyph per task. |
| `📅 YYYY-MM-DD` | Due date | Real calendar date, e.g. `📅 2026-08-01`. At most one per task. |
| `✅ YYYY-MM-DD` | Completion date | Real calendar date. At most one per task. Extracted regardless of checkbox state — optional on `[x]` tasks, accepted on `[ ]` ones. |

Other task-emoji conventions (🔁 ⏳ 🛫 ➕) are not tracker signifiers — they
are plain title text, preserved verbatim.

Task `#tags` are free-form title text, independent of the document-level
`tags` input and its registry — any token matching the character rules is
accepted and indexed; it does not need to exist in the taxonomy.

## Canonical example

```markdown
- [ ] Loose task

## Launch

- [ ] Ship pricing page #launch ⏫ 📅 2026-07-20
  Needs sign-off from Angus.
- [x] Write launch email #launch ✅ 2026-07-12

## Later

- [ ] Evaluate CDN options 🔽
```

Every signifier on one task line (also canonical):

```markdown
- [x] File taxes #finance #q3 🔺 📅 2026-07-01 ✅ 2026-06-28
```

## Common rejections

| Mistake | Error code | Fix |
| --- | --- | --- |
| `📅` followed by a non-date (`📅 tomorrow`) or nothing | `invalid_due_date` | Use a real date in `YYYY-MM-DD` format, e.g. `📅 2026-08-01`. |
| Impossible calendar day (`📅 2026-02-30`) | `invalid_due_date` | Use a date that exists on the calendar. |
| Bad or missing date after `✅` | `invalid_done_date` | Same rule: real `YYYY-MM-DD` date. |
| Two `📅` dates, two `✅` dates, or two priority glyphs on one line | `duplicate_signifier` | Keep one and delete the rest. |
| Task with signifiers but no title text (`- [ ] 📅 2026-08-01`) | `empty_task_title` | Give the task a title or remove the item. |
| Empty `##` heading with no text | `empty_heading` | Name the section (e.g. `## Launch`) or remove the heading. |
| `# Title` H1 heading in the body | `h1_heading` | Remove it — the document title lives in the title field, not the body; sections use `##`. |
| Prose paragraph before the first task or between sections | `unsupported_block` | Move it into the document description, or indent it under a task as detail. |

## Editing rules

- Make minimal diffs: change only the lines of the element being edited, and
  resend the full body with `Tatara:write_document`, always including
  `render_format: "tracker"` — the gate validates only writes that carry
  `render_format`, so a body sent without it is committed unchecked.
- Completing a task flips `[ ]` to `[x]` on one line. Rescheduling rewrites
  one `📅` value. Moving a task between sections moves exactly one list
  item — its title line plus its indented detail lines. Nothing else changes.
- On gate rejection with `canonical_body`, your change is already in those
  bytes, canonicalized — verify it landed, then resend them unmodified. On
  rejection with structured errors, fix the reported lines. Never retry the
  same body blind.

---
name: tatara-kanban
description: Authors and edits Tatara kanban board documents (render_format kanban), whose markdown must follow the canonical grammar of `## Column` headings with `- [ ]` / `- [x]` card items. Use when creating a kanban or task board in Tatara, adding or moving cards, or when a Tatara write is rejected with kanban format errors. Covers card detail lines and nested subtasks, what is disallowed (prose outside cards, H1 headings, code fences, cards before the first column), and making minimal-diff edits.
license: MIT
compatibility: Requires the Tatara MCP server connected
metadata:
  author: tatara-app
---

# Tatara kanban boards

## Format contract

- Kanban boards are markdown documents with `render_format: "kanban"`. Create
  and update them with `Tatara:write_document`, passing
  `render_format: "kanban"` on create.
- The kanban format takes no configuration in v1 — column names and order live
  in the document body. Omit `render_config`; a non-empty config is rejected.
- The write gate parses the body before saving. A body that does not parse as
  kanban is rejected with structured errors (codes below, each with a line
  number and fix instruction) plus a canonical example. A body that parses but
  is not byte-canonical is rejected with the exact corrected bytes in
  `canonical_body` — resend that, never retry blind.
- Checked `[x]` means done. Drag-and-drop in the Tatara UI rewrites this same
  markdown.

## Grammar

- Every column is a `## Heading`. Column order in the body is board order.
- Every card is a top-level `- [ ]` (open) or `- [x]` (done) list item under a
  column heading.
- Lines indented under a card are detail text, preserved verbatim.
- Indented `- [ ]` items under a card are subtasks (kept as card detail).
- Empty columns are allowed: a heading with no cards.
- Nothing else. No prose outside cards, no `#` H1 or setext headings, no code
  fences, no cards before the first column heading.

Emit canonical bytes exactly: `-` bullets, lowercase `[x]`, one blank line
after each heading and between sections. Near-miss markdown (`*`/`+` bullets,
`1.` ordered items, `###`–`######` headings, setext `---` underlines,
uppercase `[X]`, extra blank lines) parses but is rejected as non-canonical,
with the corrected `canonical_body` returned.

## Canonical example

```markdown
## Todo

- [ ] Ship pricing page
  Needs sign-off from Angus.
  - [ ] Draft copy
- [x] Write launch email

## Doing

## Done

- [x] Set up Stripe account
```

## Common rejections

| Mistake | Error code | Fix |
| --- | --- | --- |
| Cards before any column heading | `cards_before_column` | Add a `## Column` heading above them. |
| Intro prose before the first column heading | `content_before_column` | Move it into the document description. |
| `# Title` H1 heading in the body | `h1_heading` | Remove it — the document title lives in the title field, not the body; columns use `##`. |
| Setext H1 (title text over a `====` underline) | `h1_heading` | Remove it and use `## Column` headings. |
| Empty `##` heading and/or `- [ ]` with no title text | `empty_column_title`, `empty_card_title` | Name the column (e.g. `## Todo`) and give the card a title, or remove them. |
| Code fence inside a column | `unsupported_block` | Move the code into indented lines under a card, or into the document description. |
| Loose paragraph inside a column | `unsupported_block` | Indent it under a card as detail text, or move it to the document description. |

## Editing rules

- Make minimal diffs: change only the lines of the element being edited, and
  resend the full body with `Tatara:write_document`.
- Moving a card moves exactly one list item — its title line plus its indented
  detail lines — from one column section to another. Nothing else changes.
- Completing a card flips `[ ]` to `[x]` on one line. Renaming a column
  rewrites one heading line.
- On gate rejection with `canonical_body`, resend those exact bytes (with your
  intended change applied on top). On rejection with structured errors, fix
  the reported lines. Never retry the same body blind.

---
name: tatara-mindmap
description: Authors and edits Tatara mind map documents (render_format mindmap) - `## Heading` branches radiating from the document title, with nested `- ` list items as sub-topics at two spaces of indent per level. Use when creating or updating a mind map, concept map, or structured outline in Tatara, or when a write is rejected with mindmap format errors. Covers the pure nested-list form, optional `[ ]`/`[x]` checkboxes, branch paragraphs and indented topic detail, why a sub-list is a child here but detail in kanban, what is disallowed (prose outside topics, H1 headings, code fences, tables, sub-headings before the first branch), and minimal-diff subtree edits.
license: MIT
compatibility: Requires the Tatara MCP server connected
metadata:
  author: tatara-app
---

# Tatara mind maps

## Format contract

- Mind maps are markdown documents with `render_format: "mindmap"`. Create and
  update them with `Tatara:write_document`, passing `render_format: "mindmap"`
  on create.
- This skill is the body grammar only. The write workflow — search first,
  resolve the folder and tags with `Tatara:get_taxonomy`, and the fields a
  create requires (`title`, `folder`, `type`, `description`, `tags`) — is
  the core `tatara` skill; follow it alongside this one.
- The mindmap format takes no configuration in v1 — layout, collapse, and zoom
  are chosen in the UI toolbar. Omit `render_config`; a non-empty config is
  rejected.
- The write gate parses the body before saving. A body that does not parse as
  mindmap is rejected with structured errors (codes below, each with a line
  number and fix instruction) plus a canonical example. A body that parses but
  is not byte-canonical is rejected with the exact corrected bytes in
  `canonical_body` — resend that, never retry blind.
- **The document title is the root node.** The map radiates from it, so the
  body carries branches only and an H1 in the body is an error. Rename the map
  by changing the document title, not the body.
- Checked `[x]` means done, the same as in every other view. Panning, zooming,
  collapsing a branch, and switching depth are view state only — they never
  change the markdown.

## Grammar

- A body is an optional leading topic list, then zero or more `## Heading`
  branches. All three shapes are valid: a pure nested list with no headings,
  branches only, or leading topics followed by branches (leading topics come
  first). An empty body is valid.
- Every `## Heading` is a branch hanging off the root. A branch may carry one
  paragraph directly under its heading as its detail, then one nested list of
  topics. An empty branch — a bare heading — is legal.
- A topic is a `- ` list item, optionally prefixed with `[ ] ` or `[x] ` to
  make it a task. Title text may contain wikilinks, `#tags`, and emoji; all of
  it is opaque here, preserved verbatim and never interpreted.
- **A list nested under a topic is that topic's children, not its detail.**
  This is the opposite of kanban and tracker, where indented items stay with
  the card as detail text. Nesting is the point of this format.
- Non-list lines indented under a topic are its detail, preserved and shown
  under the node. Detail lines come before the child list.
- Indentation is exactly two spaces per level. Duplicate titles are legal
  anywhere — a topic is identified by its position, not its text.

Emit canonical bytes exactly: `-` bullets, lowercase `[x]`, two-space indent
per level, tight lists with no blank lines between items, one blank line after
a heading that has content, one blank line between branches. The document ends
immediately after its last content line with a single newline — a trailing
empty branch ends right after its `## Heading` line.

Near-miss markdown parses but is rejected as non-canonical, with the corrected
`canonical_body` returned: `*`/`+` bullets and `1.`/`1)` ordered items,
uppercase `[X]`, three-space, four-space or tab indentation, loose lists
(blank lines between items), setext `---` underlines, CRLF line endings,
trailing whitespace, and extra blank lines.

One conversion is worth knowing about, because it changes the shape of what
you sent: **`###` through `######` headings become topics.** A heading of
level *h* under a `##` branch canonicalizes to a topic at depth *h − 2*, its
following paragraph becomes that topic's detail, and its lists nest beneath
it. So a heading-structured document converts cleanly into a map, but expect
`canonical_body` to come back as list items.

## Canonical example

```markdown
## Ideal customer profile

- Series A–C B2B SaaS
  20–200 employees, agent-forward
- Champion: Head of Ops or CTO
  - Feels the context-rot pain weekly
  - Owns the agent budget line

## Positioning

Context infrastructure, not another wiki.

- Where we win, per [[Competitive Landscape]]
  Rivals store; we keep true
- [x] Ship the one-line positioning statement

## Risks
```

A pure nested list, with no headings at all, is equally canonical:

```markdown
- Alpha
- Beta
  - Beta child
    - Beta grandchild
- Gamma
```

## Common rejections

| Mistake | Error code | Fix |
| --- | --- | --- |
| `# Title` H1 heading in the body | `h1_heading` | Remove it — the document title is the map's root node; branches use `##`. |
| `###` or deeper before any `##` branch | `orphan_subheading` | There is no branch to attach it to. Add a `## Branch` above it, or make it `##`. |
| Empty `##` heading with no text | `empty_heading` | Name the branch (e.g. `## Risks`) or remove the heading. |
| `- ` or `- [ ]` with no title text | `empty_item_title` | Give the topic a title (e.g. `- Champion: Head of Ops`) or remove the item. |
| Intro prose before the first branch | `unsupported_block` | Move it into the document description field. |
| A second paragraph under a heading, or any paragraph after a branch's list | `unsupported_block` | A branch carries one paragraph, directly under its heading. Move the rest into the description, or indent it under a topic as detail. |
| Code fence, table, blockquote, `---` rule, or raw HTML — anywhere, including indented under a topic | `unsupported_block` | Rewrite it as plain detail lines or sub-topics, or move it into the description field. |
| A detail line starting with `-`, `*`, `+`, `1.`, or `1)` | `ambiguous_detail_line` | Rewriting the document would turn that line into a sub-topic. Make it a real sub-topic at the child indent, or rephrase it so it does not start with a marker. |

Two things that look like mistakes and are not: a `[x]` in the middle of a
title is plain text, not a checkbox (only a leading one counts), and a topic
with no checkbox at all is a normal topic rather than an unchecked task.

## Editing rules

- Make minimal diffs: change only the lines of the element being edited, and
  resend the full body with `Tatara:write_document`, always including
  `render_format: "mindmap"` — the gate validates only writes that carry
  `render_format`, so a body sent without it is committed unchecked.
- A topic and everything beneath it is one contiguous block of lines: its
  title line, its detail lines, then every more-indented line until the indent
  returns to its own level or shallower. Moving a topic moves that whole block
  and re-indents it by the depth difference. Nothing else changes.
- Promoting or demoting a topic re-indents its block by two spaces per level.
  A topic never becomes a branch by outdenting — turning a topic into a branch
  means writing a new `## Heading` and moving its block under it.
- Renaming a topic rewrites one title line; renaming a branch rewrites one
  heading line. Ticking a topic off flips `[ ]` to `[x]` on one line.
- Keep titles short — they render as nodes on a map. Put the explanation on
  the detail line beneath, not in the title.
- A title cannot start with a list marker, a checkbox, `>`, a code fence, or a
  `---` rule: it would be read back as structure rather than text. Rephrase it
  (`1. Discovery` → `Discovery`, or `Phase 1: Discovery`).
- Because the grammar is `## Section` headings plus list items, an existing
  kanban or tracker document is already a valid mind map — sending
  `render_format: "mindmap"` on an update converts it in place, with its
  checkboxes intact. Remember the inversion: subtasks that were card detail
  there become real child topics here.
- On gate rejection with `canonical_body`, your change is already in those
  bytes, canonicalized — verify it landed, then resend them unmodified. On
  rejection with structured errors, fix the reported lines. Never retry the
  same body blind.

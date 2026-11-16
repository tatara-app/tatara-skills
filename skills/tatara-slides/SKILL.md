---
name: tatara-slides
description: Authors and revises slide decks in a Tatara workspace - HTML documents with one section per slide on a fixed 1920x1080 canvas, paged by a script the workspace supplies (tatara/deck). Use when creating a presentation, pitch deck, board update, or slide deck in Tatara, when changing, adding, removing, or moving slides in one, when a write is rejected with deck issues or a base_hash mismatch, or when its render report names a slide that does not fit. Covers the slide structure, ids, speaker notes, data-bleed, reading the outline and single slides by line range, string edits anchored on a slide's opening line, the base_hash every write to a deck needs, and acting on the render report.
license: MIT
compatibility: Requires the Tatara MCP server connected
metadata:
  author: tatara-app
---

# Tatara decks

A deck is an HTML document that imports one script and keeps a fixed
structure. The general HTML rules — the sandbox, imports, images, and the
`render` result every write returns — are the sibling skill `tatara-html`;
the write workflow (search first, folder, tags) is the core `tatara` skill.
Follow both alongside this one.

Create a deck with `Tatara:create_html_document` (`folder`, `title`,
`description`, `body`, `tags`). A committed deck write returns `render` and
`deck`. `Tatara:get_taxonomy` carries the same convention, with this
example, under `file_types` → `html` → `conventions`.

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<title>Q3 update</title>
<style>
  :root { --bg: #f7f5ef; --ink: #14213d; --accent: #c2410c; }
  section { background: var(--bg); color: var(--ink); padding: 128px; font: 32px/1.4 sans-serif; }
  h1 { font-size: 96px; color: var(--accent); }
  h2 { font-size: 72px; }
</style>
<script type="module">import 'tatara/deck';</script>
</head>
<body>
<section id="cover">
  <h1>Q3 update</h1>
  <p>Board meeting</p>
  <aside>Open by thanking the board.</aside>
</section>
<section id="revenue">
  <h2>Revenue grew 18%</h2>
  <p>Retail contracts drove most of it.</p>
</section>
</body>
</html>
```

## Structure

- In `<head>`: `<script type="module">import 'tatara/deck';</script>`. It
  pages and scales the slides and checks that each one fits. Do not write
  your own paging script. Do not add a `<script type="importmap">`: the
  workspace supplies the map that resolves `tatara/deck`.
- Do not run your own render self-test in a deck: no `tatara:verify-start`,
  `state:` or `tatara:verify-done` logs. `tatara/deck` runs it, and a second
  one can end the render before the fit reports arrive, so a slide that does
  not fit comes back as fitting.
- In `<body>`: one `<section id="…">` per slide, in order, each directly
  inside `<body>`. The opening tag starts a line; `</section>` sits on a line
  of its own. Never put a section inside a slide; use `<div>`. A section
  wrapped in `<main>` or a `<div>`, or added by a script, is accepted on
  write but is not shown as a slide, and the render report says so. Match
  the render report to the outline by slide id, not by number.
- A click anywhere on a slide moves to the next one, unless it lands on a
  link, button, form field, label, summary or media element. A slide's own
  click handler (a tab, a chart) should call `event.preventDefault()` so the
  deck does not page as well.
- Ids are unique, lowercase letters, digits and hyphens, up to 64
  characters: `cover`, `revenue`, `q3-risks`. They are how you and other
  agents find a slide later. Do not rename one without a reason.
- A slide is a 1920x1080 px canvas, scaled to fit whatever shows it. Size
  everything in px, never `vw` or `vh`. Body text about 32 px, nothing under
  24 px, about 128 px of padding.
- Never set `transform`, `width`, `height`, `margin` or `position` on a
  slide's `<section>`: the script places and scales it, and the render report
  flags a slide whose box is not 1920x1080 at 0,0. Background, colour,
  padding, font, and `display: flex` or `grid` on the section are fine. Put
  layout and animation on elements inside it.
- Entrance animations are fine: the fit check finishes them before it
  measures. An animation that never ends is measured wherever it happens to
  be.
- Theme: one `<style>` block with CSS variables on `:root`. To match another
  deck, read its style lines (`deck.style_lines` in its outline) and reuse
  the variables.
- Speaker notes: at most one `<aside>` per slide. It is hidden on the slide,
  but anyone who can open the deck can read it.
- Decoration meant to run off the slide (a background shape, an image cropped
  by the edge) carries `data-bleed`; the fit check skips it and everything
  inside it. Never put `data-bleed` on text a person must read.
- One idea per slide. If a slide does not fit, shorten it or split it rather
  than shrink text below 24 px.

## Reading a deck

1. `Tatara:get_document` with `body: false`. `deck.outline` has one line per
   slide, such as `7 revenue L88-104 notes | Revenue by segment`: position,
   id, stored line range, `notes` when it has speaker notes, then the title.
   `deck.style_lines` is where the first `<style>` block sits. Keep the
   `content_hash`.
2. `Tatara:get_document` with `lines: [from, to]` for each slide you will
   change, taking the range from the outline.

Do not read the whole deck to change one slide. Line numbers move after any
write: read the outline again before reading by lines.

If the outline carries `deck.issues`, the stored deck breaks the deck rules
(it was saved some other way). Your next write lists the issues and must fix
them.

## Changing a deck

Use `Tatara:update_document` with string edits. Pass `base_hash` on every
call: the `content_hash` from your last read or write of this deck. A slide
is addressed by its opening line, `<section id="…">`, which is unique because
ids are.

- Change text: `old_string` is the text. If the same text appears on another
  slide, start `old_string` at the slide's `<section id="…">` line.
- Add a slide before the slide `asks`: `old_string` is `<section id="asks">`;
  `new_string` is the new slide, a line break, then `<section id="asks">`. To
  add at the end, do the same with `</body>`.
- Delete a slide: read its lines, send them as `old_string` with an empty
  `new_string`.
- Replace a slide: read its lines, send them as `old_string` with the new
  slide.
- Move a slide: delete it and add it where it belongs, as two edits in one
  call.

The edits in one call apply in order, each to the result of the one before,
and all or none. Each committed write returns a new `content_hash`: use it as
`base_hash` for the next write.

A committed write to a deck also returns `deck`: `slide_count`, the outline
lines of the slides this call changed or added, and the ids it `removed`.
Slides are compared by id and content, so a slide that only moved appears in
neither list.

`Tatara:write_document` with `target_document_id` and a new `body` replaces
the whole deck and needs `base_hash` too. Prefer `Tatara:update_document`.

## After every write

Read `render`. `clean` means both browsers showed the deck without errors and
every slide fits. Otherwise each entry in `render.errors` names its slide
(its `state` reads `slide 7 revenue`) and the problem. Fix them and write
again. A slide that does not fit is a broken slide: a person would see
cut-off text.

When more than four slides do not fit, the first entry is a count line that
names the rest (its `state` is `more slides`); the four furthest out follow
it. Fix those four; the next write reports the rest. `Tatara:get_document`
with `verify: true` renders the deck again without writing.

## Common rejections

Write rejections (`status: "rejected"`; nothing was written):

- `Every write to an existing deck requires base_hash: the content_hash from
  your last read or write of this document. Call get_document with body:
  false to get it. Nothing was written.` Read the outline for the current
  `content_hash` and resend with it.
- `base_hash does not match: this deck changed after you read it.` Another
  writer changed the deck; the message names the last recorded change and
  gives the current `content_hash`. Read the outline again, re-read the
  lines you were changing, and redo the edit against them. Do not resend the
  old edit blind.
- `This deck changed while this call was running.` The same, mid-call:
  re-read and redo.
- `old_string is not unique (N matches) — pass replace_all or add surrounding
  context, or include the slide's <section id="…"> line in old_string.`
- `old_string not found in body.` The text changed or its lines moved:
  re-read the slide.

Deck rules (`issues` with `rule: "deck"` and the `line`):

- `Slide sections must each start on their own line and end with </section>
  on its own line, and must not be nested.` Then one of: a `<section>` opened
  inside a slide (use `<div>`), a `<section>` tag not at the start of its
  line, a `</section>` not on a line of its own, or a slide with no closing
  `</section>`.
- `The slide starting at line N has no id.` Give every slide a unique id.
- `Slide id "…" (line N) must be lowercase letters, digits and hyphens`. The
  message suggests one.
- `Two slides share the id "…" (lines A and B).` Rename one.
- `The slide "…" has N <aside> elements. A slide has at most one <aside>: its
  speaker notes.` Merge them.
- `This document imports tatara/deck but has no <section id="…"> slides in
  <body>.`
- `A deck uses the sandbox's import map, which supplies tatara/deck. Remove
  your <script type="importmap"> and import other libraries by pinned URL.`

Render findings (in `render.errors`; the write was committed):

- `slide 7 "revenue": content extends 136px below the slide. Shorten the
  text, reduce the font size, or split the slide.` The side is one or more of
  below the slide, past the right edge, above the slide, past the left edge.
  If the offender is decoration meant to bleed, give it `data-bleed`.
- `slide 3 "pipeline" is 1920x1200, not 1920x1080. Remove the size, margin,
  position or transform your CSS sets on the slide's <section>; style
  elements inside it instead.`
- `slide 4 "chart" is not scaled to the frame: …` Your CSS sets a transform
  or transform-origin on the slide's `<section>`, which replaces the one
  that scales it. Remove it; transform elements inside the slide instead.
- `slide 5 "appendix" is not displayed. …` Remove the `hidden` attribute or
  the `display` your CSS sets on the slide's `<section>`.
- `The page's <body> does not fill the page from its top left corner, so
  every slide is moved or cut off. …` Remove the margin, border, padding,
  width or position your CSS sets on `<html>` or `<body>`.
- `1 <section> element is not shown as a slide. A slide is a <section>
  written directly inside <body> in the HTML; move the slide there.`
- `4 more slides do not fit: pipeline, outlook, asks, close.` The first
  entry when more than four slides fail. Fix the four reported after it; the
  next write reports these.

---
name: tatara-html
description: Authors and revises Tatara HTML documents (file_type html) - complete, self-contained, static HTML written with Tatara:create_html_document and rendered read-only in a sandboxed iframe. Use when creating a styled report, one-pager, or visual summary in a Tatara brain, when embedding images or charts in one, or when an HTML write is rejected with issues (incomplete, script, external_reference, size). Covers the complete-document rule, why JavaScript and external references never work, the 5 MB cap, embedding images as inline SVG or base64 data URIs, and revising via Tatara:write_document.
license: MIT
compatibility: Requires the Tatara MCP server connected
metadata:
  author: tatara-app
---

# Tatara HTML documents

## Format contract

- HTML documents are `file_type: html` — the sibling of markdown. Create them
  with `Tatara:create_html_document`: `folder`, `title`, `description`, `body`
  (the complete HTML), and `tags` (1–5) are all required. There is no `type`
  input, and no `render_format` — HTML is its own file type, not a render
  format.
- This skill is the HTML rules only. The write workflow — search first,
  resolve the folder and tags with `Tatara:get_taxonomy`, custom `frontmatter`
  keys from the field registry — is the core `tatara` skill; follow it
  alongside this one.
- The body is stored byte-verbatim and rendered read-only inside a sandboxed
  iframe in the Tatara app. A successful create returns the document's web
  `url` — share it when the user should open the document.
- Use HTML when the visual layout IS the point: a formatted report, a
  one-pager, a visual summary a teammate opens by link. For prose, notes,
  or anything a teammate should edit, write markdown with
  `Tatara:write_document` instead. And know that an HTML document is a
  static snapshot — for a live dashboard or homepage over the brain's own
  documents, use the `dashboard` render format (sibling skill
  `tatara-dashboard`) rather than hand-drawing stale numbers.

## The render sandbox — why the rules exist

The document renders in a sandboxed iframe that executes no JavaScript and
whose Content-Security-Policy blocks every network request. Nothing outside
the document's own bytes will ever load or run. Everything — styling, images,
fonts, data — must travel inside the document. The write gate rejects the
known breakers up front so you fix them before the document is ever stored.

## What the gate enforces

Every rejection returns `{ status: "rejected", reason, issues }` with one
instructive entry per broken rule — fix each and resend; never retry blind.

| Rule | What it rejects | Fix |
| --- | --- | --- |
| `incomplete` | A fragment — no `<html>`…`</html>` root | Send a complete document: `<!DOCTYPE html>`, `<html>`, `<head>`, `<body>`. |
| `script` | Any `<script>` element | Remove it. Scripts never run; interactive or script-rendered content appears broken or empty. Inline the finished result as static HTML/CSS. |
| `external_reference` | Remote URLs (`http:`, `https:`, `//…`) in resource positions: `src`, `srcset`, `poster`, `background`, `<link href>`, CSS `url()`, `@import` | Inline stylesheets into a `<style>` block; embed images and fonts as `data:` URIs. Hyperlinks (`<a href="https://…">`) are allowed — only resource loads are rejected. |
| `size` | Body over 5 MB | Recompress or resize embedded images, or split into multiple documents. |
| `empty` | An empty body | Send the document. |

Two traps the gate does NOT catch:

- A **relative or root-relative** resource URL (`src="chart.png"`,
  `src="/assets/logo.png"`) passes the gate but renders as a broken image —
  there is no file store behind the document, so the path resolves to
  nothing. Embed the bytes instead.
- `data:` URIs pass unchecked — a malformed or truncated one renders broken.
  Encode from the real bytes; never invent base64.

## Authoring rules

- The document owns its entire presentation: all CSS in `<style>` blocks or
  inline (both allowed), `<meta charset="utf-8">` in the head, system font
  stacks (embedding font files spends the byte budget fast; a `data:` URI
  font is allowed but rarely worth it).
- The iframe sizes itself to the document's height and its width follows the
  reader's pane — use responsive widths (`max-width`, percentages), not a
  fixed page width.
- Charts and diagrams are drawn, not scripted: inline `<svg>` or styled HTML
  (e.g. CSS-sized bars). A charting library is a `<script>` — rejected.
- `<a>` hyperlinks are fine, including to other Tatara documents by their web
  URL and to external sites.

## Images

Exactly two mechanisms put an image in a Tatara HTML document:

1. **Inline `<svg>` markup** — charts, diagrams, icons, logos you draw.
2. **A `data:` URI** — raster bytes (photo, screenshot, generated image)
   base64-encoded into the `src` itself.

There is no image hosting in Tatara: a brain holds markdown and HTML pages,
not image files, so there is nothing an `<img>` can point at — external URLs
are rejected and relative paths render broken. The image bytes travel inside
the document. For the full workflow — choosing the mechanism, encoding
recipes, the size budget — read [references/images.md](references/images.md).

## Revising an HTML document

`Tatara:create_html_document` is create-only. To revise, send the complete
replacement HTML as `body` to `Tatara:write_document` with
`target_document_id` — the update is a partial patch, and the new body is
validated against the same HTML rules. Omit `body` to change only metadata
(`title`, `description`, `tags`, `folder`). Do not use
`Tatara:update_document` on HTML documents — even for a one-line fix. Its
string edits are for markdown bodies and skip the HTML rules, so a slipped
edit commits broken markup silently; the replacement-body path re-validates
everything.

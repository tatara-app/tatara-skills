---
name: tatara-html
description: Authors and revises Tatara HTML documents (file_type html) - complete, self-contained, static HTML written with Tatara:create_html_document and rendered read-only in a sandboxed iframe. Use when creating a styled report, one-pager, or visual summary in a Tatara workspace, when embedding images or charts in one, or when an HTML write is rejected with issues (incomplete, script, external_reference, size). Covers the complete-document rule, why JavaScript never runs and external references are blocked (Google Fonts aside), the 5 MB cap, embedding images from workspace image assets (Tatara:upload_asset embed URLs), as inline SVG, or as base64 data URIs, and revising via Tatara:write_document.
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
  static snapshot — for a live dashboard or homepage over the workspace's own
  documents, use the `dashboard` render format (sibling skill
  `tatara-dashboard`) rather than hand-drawing stale numbers.

## The render sandbox — why the rules exist

The document renders in a sandboxed iframe that executes no JavaScript and
whose Content-Security-Policy blocks remote network requests. Two outside
sources load: a same-origin workspace-asset image
(`/api/brain/assets/<document_id>/content` — the CSP allows `img-src
'self'`), and Google Fonts (stylesheets from `fonts.googleapis.com`, font
files from `fonts.gstatic.com`). Everything else — styling, other fonts,
data, any other image — must travel inside the document's own bytes. The
write gate rejects the known breakers up front so you fix them before the
document is ever stored.

## What the gate enforces

Every rejection returns `{ status: "rejected", reason, issues }` with one
instructive entry per broken rule — fix each and resend; never retry blind.

| Rule | What it rejects | Fix |
| --- | --- | --- |
| `incomplete` | A fragment — no `<html>`…`</html>` root | Send a complete document: `<!DOCTYPE html>`, `<html>`, `<head>`, `<body>`. |
| `script` | Any `<script>` element | Remove it. Scripts never run; interactive or script-rendered content appears broken or empty. Inline the finished result as static HTML/CSS. |
| `external_reference` | Remote URLs (`http:`, `https:`, `//…`) in resource positions: `src`, `srcset`, `poster`, `background`, `<link href>`, CSS `url()`, `@import` | Inline stylesheets into a `<style>` block. For images, reference workspace assets by their stable embed URL (upload via `Tatara:upload_asset`), or embed small one-offs as `data:` URIs. Google Fonts (`fonts.googleapis.com`, `fonts.gstatic.com`) is the one allowed remote source. Hyperlinks (`<a href="https://…">`) are allowed — only resource loads are rejected. |
| `size` | Body over 5 MB | Recompress or resize embedded images, or split into multiple documents. |
| `empty` | An empty body | Send the document. |

Three traps the gate does NOT catch:

- A **made-up relative path** (`src="chart.png"`, `src="/images/logo.png"`)
  passes the gate but renders as a broken image — the only path that serves
  image bytes is the asset embed form `/api/brain/assets/<document_id>/content`,
  with an id returned by `Tatara:upload_asset` or `Tatara:get_asset`.
- A **signed/temporary URL** (`download_url` from `Tatara:get_asset`, a
  storage or generation-service URL) is remote, so the gate rejects it — but
  even same-origin expiring links must never be stored: always embed the
  stable `embed_url`.
- `data:` URIs pass unchecked — a malformed or truncated one renders broken.
  Encode from the real bytes; never invent base64.

## Authoring rules

- The document owns its entire presentation: all CSS in `<style>` blocks or
  inline (both allowed), `<meta charset="utf-8">` in the head, system font
  stacks or Google Fonts (embedding font files spends the byte budget fast;
  a `data:` URI font is allowed but rarely worth it).
- The iframe sizes itself to the document's height and its width follows the
  reader's pane — use responsive widths (`max-width`, percentages), not a
  fixed page width.
- Charts and diagrams are drawn, not scripted: inline `<svg>` or styled HTML
  (e.g. CSS-sized bars). A charting library is a `<script>` — rejected.
- `<a>` hyperlinks are fine, including to other Tatara documents by their web
  URL and to external sites.

## Images

Exactly three mechanisms put an image in a Tatara HTML document:

1. **A workspace image asset** — the preferred route for real rasters (photos,
   screenshots, generated images). Store the image with
   `Tatara:upload_asset` (or find an existing one), then reference its
   stable `embed_url`: `<img src="/api/brain/assets/<document_id>/content">`.
   Assets are governed documents (`file_type: image`) in the folder tree,
   and their bytes do not count against this document's 5 MB.
2. **Inline `<svg>` markup** — charts, diagrams, icons, logos you draw.
3. **A `data:` URI** — small one-off raster bytes base64-encoded into the
   `src` itself, when a governed, reusable asset isn't warranted.

Any other reference breaks: external URLs are rejected by the gate, and
invented relative paths render broken. For the full workflow — the asset
tools, choosing the mechanism, encoding recipes, the size budget — read
[references/images.md](references/images.md).

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

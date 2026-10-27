# Embedding images in Tatara HTML documents

The render sandbox blocks every remote image request, and the write gate
rejects remote URLs in resource positions. Three mechanisms render — a workspace
image asset referenced by its stable embed URL, inline SVG markup, or a
`data:` URI. Everything else is rejected or silently broken.

## Choosing the mechanism

| The image is… | Use |
| --- | --- |
| A real raster — photo, screenshot, ad creative, generated image | A **workspace image asset**: `Tatara:upload_asset`, then its `embed_url` |
| Already in the workspace as an image asset | Its `embed_url` (find it with `Tatara:search_documents`; fetch URLs with `Tatara:get_asset`) |
| A chart, diagram, icon, or illustration you can draw | Inline `<svg>` — no encoding step, crisp at any zoom, tiny |
| A tiny one-off raster nobody will reuse | A `data:` URI |
| Referenced by URL and you cannot obtain it any way | Don't fake it — draw an SVG placeholder or omit it, and tell the user why |

## Workspace image assets — the preferred route

Images stored in the workspace are governed **asset documents**: `file_type:
image`, filed in the folder tree with a title, description, and tags,
findable with `Tatara:search_documents` / `Tatara:list_documents` like any
document. The bytes live in managed storage and stream from a stable,
same-origin embed URL that the render sandbox loads (`img-src 'self'`).

Store an image with `Tatara:upload_asset` (create-only; `folder`, `title`,
`description`, `tags` 1–5 required, like other creates):

- Send the bytes by reference — `source_url` (an https URL, e.g. a
  generation service's output) is fetched server-side, so the image never
  transits your context — or inline as `data_base64` for small files.
  Exactly one of the two.
- Formats: PNG, JPEG, WebP, GIF — sniffed from the bytes, up to 10 MB. No
  SVG (draw SVG inline instead).
- The result carries `embed_url` — reference it verbatim:

```html
<img alt="March cohort dashboard"
     src="/api/brain/assets/2f6c…e91/content"
     style="width:100%;max-width:640px;height:auto">
```

For an existing asset, `Tatara:get_asset` returns the same stable
`embed_url` plus a short-lived signed `download_url` for pulling the bytes.
**Only `embed_url` ever goes into a document** — a signed URL expires in
minutes and the image would break.

Why prefer assets over `data:` URIs: the bytes don't count against the HTML
document's 5 MB cap, the image is reusable across documents, and it's
governed — described, tagged, searchable, recoverable. An asset's content is
immutable; to revise an image, upload a new asset and update the references.

## Inline SVG

Prefer this for anything you are drawing yourself — bar charts, flow
diagrams, sparklines, logos, icons. It is markup, so it needs no encoding,
scales losslessly, and costs a fraction of the bytes of a raster.

```html
<svg viewBox="0 0 320 120" role="img" aria-label="Signups per month"
     style="width:100%;max-width:480px">
  <rect x="10"  y="76" width="80" height="40" fill="#4a6b5d"/>
  <rect x="120" y="49" width="80" height="67" fill="#4a6b5d"/>
  <rect x="230" y="20" width="80" height="96" fill="#4a6b5d"/>
  <text x="50"  y="112" text-anchor="middle" font-size="10">Jan 400</text>
  <text x="160" y="112" text-anchor="middle" font-size="10">Feb 650</text>
  <text x="270" y="112" text-anchor="middle" font-size="10">Mar 910</text>
</svg>
```

Rules inside the SVG: no `<script>` (the gate scans the whole body), no
remote references (`<image href="https://…">`, remote fonts in SVG CSS —
both rejected), size with `viewBox` plus CSS width so it scales.

## data: URIs

The fallback for a small raster that doesn't warrant a governed asset.
Syntax: `data:<mime-type>;base64,<encoded bytes>` in any resource position —
`<img src>`, `srcset`, or CSS `background-image: url(…)`. Mime types:
`image/png`, `image/jpeg`, `image/webp`, `image/gif`, `image/svg+xml`.

Encode the real file — never invent base64 (a fabricated or truncated URI
passes the gate and renders broken):

```bash
# POSIX
base64 -w0 photo.jpg          # macOS: base64 -i photo.jpg | tr -d '\n'
```

```powershell
# PowerShell
[Convert]::ToBase64String([IO.File]::ReadAllBytes('photo.jpg'))
```

Then: `<img alt="…" src="data:image/jpeg;base64,/9j/4AAQ…" style="width:100%;max-width:640px;height:auto">`.

Always set `alt`, and constrain display size with CSS — the intrinsic pixel
size of an embedded image is often far larger than the pane.

## The size budget

- The HTML document — markup, CSS, and every `data:` URI — must stay under
  **5 MB**. Base64 inflates bytes by ~33%, so roughly 3.5 MB of raw inlined
  image data is the practical ceiling. The gate names the document's actual
  size in its `size` rejection.
- **Asset-embedded images cost the document almost nothing** — each asset
  has its own 10 MB limit and lives outside the body. An image-heavy report
  should use assets, keeping the document itself small.
- Before inlining a raster as a `data:` URI, resize to display size
  (1200–1600 px wide is plenty) and recompress — JPEG/WebP quality ~80 for
  photos; PNG only for flat-color UI captures and transparency. One
  `magick photo.png -resize 1600x -quality 80 photo.jpg` (ImageMagick)
  routinely turns a 4 MB capture into 200 KB.

## Common failures

| Attempt | Outcome | Instead |
| --- | --- | --- |
| `<img src="https://…/photo.png">` | Rejected — `external_reference` | `Tatara:upload_asset` with that URL as `source_url`, then embed the returned `embed_url` |
| Embedding `get_asset`'s `download_url` | Rejected (remote URL) — and it expires in minutes anyway | Embed the stable `embed_url` |
| `<img src="chart.png">` or an invented path | Passes the gate, renders broken — only the asset embed form serves bytes | Upload the image as an asset, or draw it inline |
| Uploading an SVG via `upload_asset` | Rejected — SVG is not a supported asset format | Put the SVG markup inline in the document |
| Inventing a base64 string | Passes the gate, renders broken | Encode the real file |
| Inlining a 4 MB screenshot as a `data:` URI | Rejected — `size` (base64 inflation) | Upload it as an asset and embed the `embed_url`, or resize + recompress first |
| Chart via a charting library | Rejected — `script` | Draw the chart as inline SVG |

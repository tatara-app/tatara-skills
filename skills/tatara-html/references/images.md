# Embedding images in Tatara HTML documents

Every image must live inside the document itself. The render sandbox blocks
all network requests, the write gate rejects remote URLs in resource
positions, and a brain holds no image files a path could resolve to. Two
mechanisms render; everything else is rejected or silently broken.

## Choosing the mechanism

| The image is… | Use |
| --- | --- |
| A chart, diagram, icon, logo, or illustration you can draw | Inline `<svg>` — no encoding step, crisp at any zoom, tiny |
| A photo, screenshot, or generated raster you have the bytes of | `data:` URI |
| Referenced by URL, but you can fetch it | Fetch the bytes, then embed as a `data:` URI |
| Referenced by URL and you cannot fetch it | Don't fake it — draw an SVG placeholder or omit it, and tell the user why |

Never invent base64. A fabricated or truncated `data:` URI passes the gate
and renders as a broken image.

## Inline SVG

Prefer this for anything you are generating yourself — bar charts, flow
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

Syntax: `data:<mime-type>;base64,<encoded bytes>` in any resource position —
`<img src>`, `srcset`, or CSS `background-image: url(…)`.

Mime types: `image/png`, `image/jpeg`, `image/webp`, `image/gif`,
`image/svg+xml`.

Encode the real file:

```bash
# POSIX
base64 -w0 photo.jpg          # macOS: base64 -i photo.jpg | tr -d '\n'
```

```powershell
# PowerShell
[Convert]::ToBase64String([IO.File]::ReadAllBytes('photo.jpg'))
```

```js
// Node
fs.readFileSync('photo.jpg').toString('base64')
```

Then:

```html
<img alt="March cohort dashboard"
     src="data:image/jpeg;base64,/9j/4AAQSkZJRg…"
     style="width:100%;max-width:640px;height:auto">
```

Always set `alt`, and constrain display size with CSS — the intrinsic pixel
size of an embedded image is often far larger than the pane.

## The size budget

The whole document — markup, CSS, and every embedded image — must stay under
**5 MB**. Base64 inflates bytes by ~33%, so roughly **3.5 MB of raw image
data is the practical ceiling**, shared across all images. The gate names the
document's actual size in its `size` rejection.

Before encoding, resize and recompress:

- Resize to display size — 1200–1600 px wide is plenty for a full-width
  figure; a screenshot never needs its Retina pixel dimensions.
- Photos and screenshots with gradients: JPEG or WebP, quality ~80. PNG only
  for flat-color UI captures and images that need transparency.
- One `magick photo.png -resize 1600x -quality 80 photo.jpg` (ImageMagick)
  before encoding routinely turns a 4 MB capture into 200 KB. Any image tool
  works; the point is: never embed original bytes unexamined.

Over budget after compressing? Split the report into multiple documents and
hyperlink between them — links to other Tatara documents by web URL are
allowed.

## Common failures

| Attempt | Outcome | Instead |
| --- | --- | --- |
| `<img src="https://…/logo.png">` | Rejected — `external_reference` | Fetch the bytes and embed as a `data:` URI, or draw it as SVG |
| `<img src="chart.png">` / `/assets/x.png` | Passes the gate, renders broken — nothing serves that path | Embed the bytes |
| Pointing `<img>` at another Tatara document's URL | Rejected (remote URL) — and documents are pages, not image files | Embed the bytes |
| Inventing a base64 string | Passes the gate, renders broken | Encode the real file |
| Embedding a 4 MB screenshot raw | Rejected — `size` (base64 inflation) | Resize + recompress first |
| Chart via a charting library | Rejected — `script` | Draw the chart as inline SVG |

---
name: tatara-dashboard
description: Authors and edits Tatara dashboard documents (render_format dashboard) - a single dashboard-fenced YAML code block composing rows of live query blocks (stat, ratio, list, table, cards, bar) and curated blocks (links, prose, callout) into a brain homepage. Use when creating or updating a dashboard, homepage, overview, or home document in Tatara, or when a write is rejected with dashboard format errors. Covers the row and block layout (1-3 blocks per row), the shared base filter grammar, per-block keys and defaults, summarize ops, and the rule that nothing may appear outside the block.
license: MIT
compatibility: Requires the Tatara MCP server connected
metadata:
  author: tatara-app
---

# Tatara dashboards

## Format contract

- Dashboards are markdown documents with `render_format: "dashboard"` — a
  composed, read-only homepage over the other documents in a brain. Every
  query block re-runs live against the current documents; the dashboard
  shows the brain as it is now, not as it was when written. Create and
  update them with `Tatara:write_document`, passing
  `render_format: "dashboard"` on create.
- This skill is the body grammar only. The write workflow — search first,
  resolve the folder and tags with `Tatara:get_taxonomy`, and the fields a
  create requires (`title`, `folder`, `type`, `description`, `tags`) — is
  the core `tatara` skill; follow it alongside this one.
- A dashboard keeps all configuration inside its body. Omit
  `render_config`; a non-empty config is rejected with an instruction to
  resend without it.
- The body is exactly one ` ```dashboard ` fenced code block containing a
  YAML mapping with top-level keys `filters` (optional) and `rows`
  (required). Nothing else — no prose before or after the block (context
  belongs in the document description field; welcome text belongs in a
  `prose` block), no second block.
- The write gate parses the body before saving. A body that does not parse
  as dashboard is rejected with structured errors (codes below, each with a
  fix instruction) plus a canonical example. A body that parses but is not
  byte-canonical is rejected with the exact corrected bytes in
  `canonical_body` — resend that, never retry blind.

Emit canonical bytes exactly: LF line endings; the opening
` ```dashboard ` line first, the closing ` ``` ` fence as the last line of
the body, followed by a single newline; 2-space indent; block-style lists
and mappings only; fixed key order — top level `filters`, `rows`; each
block `type`, `title`, `variant`, `body`, `filters`, `numerator`,
`denominator`, `groupBy`, `columns`, `sort`, `column`, `summarize`,
`limit`, `href`, `items` (only the keys the block uses, in that order).
Filter expression strings are always single-quoted, with an embedded `'`
doubled as `''`, and always on their own indented line under their key —
never inline, never a `- ` list item. `body` text is always emitted as a
`|-` block scalar; its content is normalized on write (CRLF to LF,
trailing whitespace stripped, common leading indent removed, no leading or
trailing blank lines). Every other scalar is written plain unless YAML
would misread it, in which case it is single-quoted.

Near-miss YAML (flow-style `[a, b]` / `{k: v}`, keys in a different order,
inline or double-quoted filter expressions, a plain-string `body`, CRLF
line endings, blank lines around the block) parses but is rejected as
non-canonical, with the corrected `canonical_body` returned.

## Canonical example

````markdown
```dashboard
filters:
  '!file.inFolder("archive")'
rows:
  - blocks:
      - type: stat
        title: Documents
      - type: stat
        title: Needs review
        filters:
          'last_reviewed < now() - "30 days"'
        href: ops/review-queue
      - type: stat
        title: Est. ARR ($M)
        filters:
          'type == "competitor"'
        summarize:
          property: arr
          op: Sum
  - blocks:
      - type: ratio
        title: Docs with an owner
        numerator:
          '!owner.isEmpty()'
      - type: bar
        title: Docs by type
        groupBy: type
        limit: 6
  - blocks:
      - type: list
        title: Recently updated
        limit: 6
      - type: cards
        title: Playbooks
        filters:
          'file.hasTag("playbook")'
        limit: 4
  - blocks:
      - type: table
        title: Competitors
        filters:
          'type == "competitor"'
        columns:
          - title
          - tier
          - arr
          - last_reviewed
        sort:
          - property: tier
            direction: ASC
        limit: 10
  - blocks:
      - type: links
        title: Start here
        items:
          - label: Our ICP
            to: strategy/icp
          - label: Pricing
            to: sales/pricing
      - type: callout
        title: Keep it true
        variant: tip
        body: |-
          Verify [[Pricing]] after any plan change — this dashboard reads it live.
      - type: prose
        body: |-
          Welcome to the brain. Every number here is a live query over the
          documents; edit this dashboard by editing its YAML.
```
````

The minimal valid dashboard — one row with one block:

````markdown
```dashboard
rows:
  - blocks:
      - type: stat
        title: Documents
```
````

## Layout

- `filters` (optional): a global filter — one expression string or an
  `and:` / `or:` / `not:` tree — ANDed into **every** query block. Use it
  to scope a whole dashboard ("only the sales folder").
- `rows` (required): a non-empty list. Each row is `blocks:` holding 1–3
  blocks, rendered as that many columns. Full-width blocks (`table`) read
  best alone in a row; stats read best three-up.

## Block reference

Query blocks (`stat`, `ratio`, `list`, `table`, `cards`, `bar`) filter the
brain's documents with the same expression grammar as the `base` format —
see the sibling skill `tatara-base` for the full grammar (comparisons,
`&&` `||` `!`, `.contains()`, `.isEmpty()`, `file.hasTag()`,
`file.inFolder()`, `date()` / `now()` / `today()`, and date ± `"duration"`
arithmetic). Property references are also base's: a bare name (universal
column or frontmatter key), `note.<key>`, or `file.name` / `file.ctime` /
`file.mtime` / `file.folder` / `file.tags`. All query blocks accept an
optional `href` — a brain document path rendered as a "See all" link.

### `stat` — one number

| Key | Required | Value |
| --- | --- | --- |
| `title` | yes | Label above the number. |
| `filters` | no | Which documents count; omitted = all in scope. |
| `summarize` | no | `property:` + `op:` — aggregate that property over matching docs instead of counting them. Ops are base's case-sensitive summary names: `Empty`, `Filled`, `Unique`, `Sum`, `Average`, `Min`, `Max`, `Median`, `Range`, `Stddev`, `Earliest`, `Latest`, `Checked`, `Unchecked`. |
| `href` | no | Doc path for the See-all link. |

### `ratio` — share of the brain, with a meter

| Key | Required | Value |
| --- | --- | --- |
| `title` | yes | Label. |
| `numerator` | yes | Filter (string or `and`/`or`/`not` tree) for the matching subset. |
| `denominator` | no | Filter for the comparison population. Omitted = every document in scope (global `filters` still apply). |
| `href` | no | See-all link. |

### `list` — compact document rows

| Key | Required | Value |
| --- | --- | --- |
| `title` | yes | Label. |
| `filters` | no | Which documents. |
| `sort` | no | List of `- property:` / `direction: ASC\|DESC` pairs. Default: `updated` DESC (a recency list). |
| `column` | no | One property ref shown right-aligned per row. Default `updated`, rendered as relative time. |
| `limit` | no | Max rows. Default 8. |
| `href` | no | See-all link. |

### `table` — embedded read-only base table

| Key | Required | Value |
| --- | --- | --- |
| `title` | yes | Label. |
| `filters` | no | Which documents. |
| `columns` | no | Property refs, left to right. Default `title`, `type`, `tags`, `updated`. |
| `sort` | no | Same shape as list. Default `title` ASC. |
| `limit` | no | Max rows. Default 10. |
| `href` | no | See-all link. |

### `cards` — title/description/tags grid

Same keys as `list` minus `column`. Default sort `updated` DESC, default
`limit` 6.

### `bar` — top groups by count

| Key | Required | Value |
| --- | --- | --- |
| `title` | yes | Label. |
| `filters` | no | Which documents. |
| `groupBy` | yes | One property ref (a plain string, not a mapping) — documents are bucketed by its value. |
| `limit` | no | Max bars shown; the top groups by count. Default 8. Groups beyond it collapse to "+N more". |
| `href` | no | See-all link. |

### `links` — curated navigation

| Key | Required | Value |
| --- | --- | --- |
| `title` | yes | Label. |
| `items` | yes | Non-empty list of `- to: <doc path>` with optional `label:` (emitted before `to`). A missing label falls back to the target document's title, then to the path. |

### `prose` — free text

One key, `body` (required): text emitted as a `|-` block scalar.
`[[wikilinks]]` resolve against document titles and render as links. No
`title` key — use `callout` for titled text.

### `callout` — emphasized text

| Key | Required | Value |
| --- | --- | --- |
| `title` | no | Small heading inside the callout. |
| `variant` | no | `note` (default), `warning`, or `tip`. |
| `body` | yes | Same rules as `prose`. |

## Common rejections

| Mistake | Error code | Fix |
| --- | --- | --- |
| No ` ```dashboard ` block in the body | `missing_dashboard_block` | Send exactly one fenced block: ` ```dashboard ` … ` ``` ` containing the YAML layout. |
| Prose before or after the block | `prose_outside_block` | Move it into a `prose` or `callout` block inside the dashboard. |
| More than one ` ```dashboard ` block | `multiple_dashboard_blocks` | Merge the rows into a single block. |
| Block opened but never closed | `unclosed_dashboard_block` | End the block with a line containing only ` ``` `. |
| YAML syntax error inside the block | `invalid_yaml` | Fix the YAML; the error message carries the parser detail. |
| Unknown block type, a row with 0 or 4+ blocks, missing required keys, unknown keys | `invalid_config` | Only the nine block types exist, shaped as in the block reference; each row holds 1–3 blocks. |
| Bad filter expression (in `filters`, `numerator`, or `denominator`) | `invalid_expression` | Use the base filter grammar; the message names the offending block as `rows[i].blocks[j]`. |
| Malformed property reference in `columns`, `sort`, `column`, `groupBy`, or `summarize` | `invalid_property_ref` | A bare name, `note.<key>`, or `file.<prop>`. |
| Unknown `summarize` op (`sum` instead of `Sum`) | `unknown_summary` | Use one of the case-sensitive summary names listed under `stat`. |
| Empty or whitespace-only `body` on `prose` / `callout` | `empty_body` | Give the block text content, or remove it. |

## Editing rules

- Make minimal diffs: change only the YAML lines being edited, and resend
  the full body with `Tatara:write_document`, always including
  `render_format: "dashboard"` — the gate validates only writes that carry
  `render_format`, so a body sent without it is committed unchecked.
- Adding a block appends one `- type: …` item to a row's `blocks` (max 3
  per row — add a new row when full). Adding a row appends one
  `- blocks:` item to `rows`. Retitling a block edits one `title` line.
  Tightening a query edits only that block's `filters` node. Defaults
  (list/cards/table limits and sorts, bar's top-8) are render-side — never
  write them into the YAML unless changing them.
- On gate rejection with `canonical_body`, your change is already in those
  bytes, canonicalized — verify it landed, then resend them unmodified. On
  rejection with structured errors, fix the reported blocks (`rows[i].blocks[j]`
  addresses point into your YAML). Never retry the same body blind.

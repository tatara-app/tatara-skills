---
name: tatara-base
description: Authors and edits Tatara base documents (render_format base) - a single base-fenced YAML code block defining filters, properties, and views over the documents in a brain, Tatara's saved-view and index format. Use when creating or updating a base, saved view, or document index in Tatara, or when a write is rejected with base format errors. Covers filter expressions and functions, property types (text, number, date, select with options), views (table type, order, sort, groupBy, limit, summaries), and the rule that nothing may appear outside the block.
license: MIT
compatibility: Requires the Tatara MCP server connected
metadata:
  author: tatara-app
---

# Tatara bases

## Format contract

- Bases are markdown documents with `render_format: "base"` — live table
  views over the other documents in a brain. Rows are documents; columns are
  universal properties (`title`, `description`, `type`, `tags`, `source`,
  `created`, `updated`, `folder`) and frontmatter keys. Create and update
  them with `Tatara:write_document`, passing `render_format: "base"` on
  create.
- A base keeps all configuration inside its body. Omit `render_config`; a
  non-empty config is rejected with an instruction to resend without it.
- The body is exactly one ` ```base ` fenced code block containing a YAML
  mapping with top-level keys `filters`, `properties`, `views` (only
  `views` is required). Nothing else
  — no prose before or after the block (context belongs in the document
  description field), no second block.
- The write gate parses the body before saving. A body that does not parse
  as base is rejected with structured errors (codes below, each with a fix
  instruction) plus a canonical example. A body that parses but is not
  byte-canonical is rejected with the exact corrected bytes in
  `canonical_body` — resend that, never retry blind.

Emit canonical bytes exactly: LF line endings; the opening ` ```base ` line
first, the closing ` ``` ` fence as the last line of the body, followed by a
single newline; 2-space indent; block-style lists and mappings only; fixed
key order — top level `filters`, `properties`, `views`; each property
`type`, `displayName`, `options`; each view `type`, `name`, `filters`,
`order`, `sort`, `groupBy`, `limit`, `summaries`. Filter expression strings
are always single-quoted, with an embedded `'` doubled as `''` — the
expression `title.contains("it's")` is written `'title.contains("it''s")'`;
every other scalar is written plain unless YAML would misread it, in which
case it is single-quoted. A lone filter expression is emitted as a plain
scalar on its own indented line under `filters:` — never inline after the
key, and never as a `- ` list item.

Near-miss YAML (flow-style `[a, b]` / `{k: v}`, keys in a different order,
double-quoted or inline filter expressions, CRLF line endings, blank lines
before, after, or inside the block) parses but is rejected as
non-canonical, with the corrected `canonical_body` returned.

## Canonical example

````markdown
```base
filters:
  and:
    - 'type == "competitor"'
    - '!tier.isEmpty()'
properties:
  title:
    displayName: Competitor
  tier:
    type: select
    options:
      - leader
      - challenger
      - niche
  arr:
    type: number
    displayName: Est. ARR ($M)
  last_reviewed:
    type: date
views:
  - type: table
    name: All competitors
    order:
      - title
      - tier
      - arr
      - last_reviewed
      - tags
    sort:
      - property: tier
        direction: ASC
    limit: 50
    summaries:
      arr: Sum
  - type: table
    name: Needs review
    filters:
      and:
        - 'last_reviewed < now() - "30 days"'
    groupBy:
      property: tier
      direction: ASC
```
````

The minimal valid base — `views` with one table view is the only required
content:

````markdown
```base
views:
  - type: table
    name: All
```
````

## Key reference

### `filters` (optional)

A single expression string, or a tree of `and:` / `or:` / `not:` nodes,
each holding a non-empty list of expression strings or nested nodes. `not`
matches rows that match none of its children. Top-level filters apply to
every view; a view's own `filters` (same shape) narrows further — a row
must pass both.

### `properties` (optional)

A mapping keyed by property reference; each entry declares how that column
is presented (all keys optional):

| Key | Value |
| --- | --- |
| `type` | One of `text`, `number`, `checkbox`, `date`, `datetime`, `select`, `list`. |
| `displayName` | Column header label. |
| `options` | Non-empty list of strings. `type: select` and `options` go together — declare both or neither. |

Property references — used as `properties` keys and in `order`, `sort`,
`groupBy`, and `summaries`:

- A bare name (`tier`) — a universal property if the name matches one,
  otherwise a frontmatter key.
- `note.<key>` — a frontmatter key, explicitly.
- `file.name`, `file.ctime`, `file.mtime`, `file.folder`, `file.tags` —
  aliases for `title`, `created`, `updated`, `folder`, `tags`.

Anything else (`a.b.c`) is rejected. Undeclared columns still work:
universal properties have built-in types (`tags` is list,
`created`/`updated` are datetime, the rest text) and frontmatter values get
an inferred type. Declare a property only to set its header, force a type,
or define select options.

### `views` (required, at least one)

| Key | Required | Value |
| --- | --- | --- |
| `type` | yes | Always `table`. |
| `name` | yes | Tab label; unique across views. |
| `filters` | no | Same shape as top-level filters; combined with them. |
| `order` | no | Non-empty list of property refs — the columns, left to right. Default columns: `title`, `type`, `tags`, `updated`. |
| `sort` | no | List of `- property: <ref>` / `direction: ASC` (or `DESC`) pairs, applied in order. Default: `title` ascending. Rows missing the value sort last. |
| `groupBy` | no | One `property:` / `direction:` pair — groups rows into sections. |
| `limit` | no | Positive integer — max rows shown after filtering and sorting. |
| `summaries` | no | Mapping of property ref to summary name, shown under the column. |

Summary names are case-sensitive: `Empty`, `Filled`, `Unique` (any column);
`Sum`, `Average`, `Min`, `Max`, `Median`, `Range`, `Stddev` (number
columns); `Earliest`, `Latest`, `Range` (date/datetime columns); `Checked`,
`Unchecked` (checkbox columns). A summary that does not fit the column's
type displays as `—`.

## Filter expressions

Every expression string is compiled at write time; anything outside this
grammar is rejected with `invalid_expression`:

- Comparisons: `==` `!=` `>` `<` `>=` `<=`
- Logic: `!` `&&` `||`, parentheses for grouping
- Literals: quoted strings (single or double quotes; no escapes — a string
  cannot contain its own quote character), non-negative numbers (there is
  no unary minus), `true` / `false`
- Property paths: a bare name, `note.<key>`, or `file.<prop>` (`file.name`,
  `file.ctime`, `file.mtime`, `file.folder`, `file.tags`). Bracket access
  (`x["y"]`) is not supported.
- Globals: `date("YYYY-MM-DD")`, `now()`, `today()` (midnight today).
  Argument counts are enforced.
- Value methods: `.contains(x)` (substring on strings, membership on
  lists), `.isEmpty()` (missing, empty string, or empty list)
- File methods: `file.hasTag("tag")`, `file.inFolder("path")` (the folder
  itself or any subfolder)
- Date arithmetic: date ± `"duration"`, e.g. `now() - "30 days"`. A
  duration string is an integer plus a unit: `y`/`year`/`years`,
  `M`/`month`/`months`, `w`/`week`/`weeks`, `d`/`day`/`days`,
  `h`/`hour`/`hours`, `m`/`minute`/`minutes`, `s`/`second`/`seconds`
  (lowercase `m` is minutes; months are `M`). `*` and `/` have no meaning
  in filters.

No other functions or methods exist — `file.hasLink()`, for example, is
rejected. There are no formulas anywhere in a base document: a top-level
`formulas:` key is rejected as `formulas_unsupported`.

Evaluation never errors on a row: a missing property is null, null fails
every comparison, and mismatched types compare false — with one exception,
an ISO date string (`2026-07-01`) coerces when compared against a date from
`date()` / `now()` / `today()` / date arithmetic. A bare value expression
is a truthiness test (missing, empty string, `0`, `false`, and empty lists
are all false).

## Common rejections

| Mistake | Error code | Fix |
| --- | --- | --- |
| No ` ```base ` block in the body | `missing_base_block` | Send exactly one fenced block: ` ```base ` … ` ``` ` containing the YAML definition. |
| Prose before or after the block | `prose_outside_block` | Move it into the document description field. |
| More than one ` ```base ` block | `multiple_base_blocks` | Merge the definitions, or split into separate documents. |
| Block opened but never closed | `unclosed_base_block` | End the block with a line containing only ` ``` `. |
| YAML syntax error inside the block | `invalid_yaml` | Fix the YAML; the error message carries the parser detail. |
| Unknown or misshapen keys, missing `views`, top level not a mapping | `invalid_config` | Only `filters`, `properties`, `views` exist, shaped as in the key reference. |
| Top-level `formulas:` key | `formulas_unsupported` | Remove it — formulas are not part of base documents. |
| Unknown function or bad expression syntax (`file.hasLink("x")`) | `invalid_expression` | Use only the filter-expression grammar above. |
| Malformed property reference (`a.b.c`) | `invalid_property_ref` | A bare name, `note.<key>`, or `file.<prop>`. |
| `type: select` without `options`, or `options` without `type: select` | `select_options` | Declare both or neither. |
| Two views sharing a name | `duplicate_view_name` | Give each view a unique name — it is the tab label. |
| Unknown summary name (`sum` instead of `Sum`) | `unknown_summary` | Use one of the case-sensitive summary names listed above. |

## Editing rules

- Make minimal diffs: change only the YAML lines being edited, and resend
  the full body with `Tatara:write_document`.
- Adding a column adds one line to a view's `order`, plus a `properties`
  entry only if it needs a header, type, or options. A view without `order`
  first needs the full list written out — the default columns `title`,
  `type`, `tags`, `updated` — plus the new one, or the view collapses to
  just the columns listed. Adding a view appends
  one `- type: table` item to `views`. Retitling a column edits one
  `displayName` line. Tightening a view edits only its `filters` node.
- On gate rejection with `canonical_body`, your change is already in those
  bytes, canonicalized — verify it landed, then resend them unmodified. On
  rejection with structured errors, fix the reported issues. Never retry
  the same body blind.

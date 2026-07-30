---
name: tatara-table
description: Authors and edits Tatara table documents (render_format table) - a single GitHub-flavored pipe table with typed columns declared in render_config (text, number, checkbox, date, select with options) and optional formula columns. Use when creating or updating a data table, grid, or spreadsheet-like document in Tatara, or when a write is rejected with table format errors. Covers header rules, per-type cell validity, blank cells, and formula behavior (computed values are materialized into the table).
license: MIT
compatibility: Requires the Tatara MCP server connected
metadata:
  author: tatara-app
---

# Tatara data tables

## Format contract

- Tables are markdown documents with `render_format: "table"`. Create and
  update them with `Tatara:write_document`, passing `render_format: "table"`
  (plus `render_config` for typed columns) on create.
- The body is exactly one GFM pipe table: a header row, a separator row, then
  data rows. Nothing else — no prose before or after (context belongs in the
  document description field), no second table. A header-only table with zero
  data rows is valid.
- `render_config.columns` declares column types, keyed by exact header text;
  a column with no entry is text. An all-text table needs no `render_config`
  at all. Optional `sort` and `filters` entries are non-destructive view
  defaults — they never reorder or drop the markdown rows. An invalid config
  (unknown type, `select` without options, unknown keys) is rejected with the
  schema problem and an example config.
- The write gate parses the body before saving. A body that does not parse as
  table is rejected with structured errors (codes below, each with a fix
  instruction; cell and header errors carry a line number) plus a canonical
  example. A body that parses but is not byte-canonical is rejected with the
  exact corrected bytes in `canonical_body` — resend that, never retry blind.
- On update, an omitted `render_format` preserves the format and its config.
  To change column types, options, or formulas, resend
  `render_format: "table"` with the full new `render_config` in the same
  call; the sent config replaces the stored one wholesale, and
  `render_config` is never accepted without `render_format`. The gate then
  re-validates the document's body against the new config.

Emit canonical bytes exactly: every row starts and ends with `|`, one space
around each cell value, an empty cell is a single space between its pipes,
separator cells are exactly `---` with no alignment colons, no blank lines.
A literal `|` inside a cell is written `\|` — the only escape. The document
ends immediately after the last row with a single newline.

Near-miss tables (alignment colons `:---` / `---:`, extra cell padding,
missing edge pipes, ragged rows, blank checkbox cells, uncomputed or stale
formula cells) parse but are rejected as non-canonical, with the corrected
`canonical_body` returned. Ragged rows canonicalize to the header width:
short rows pad with blank cells, cells beyond the header width are dropped.

## Canonical example

Body:

```markdown
| Supplier | Status | Renewal | Active | Cost |
| --- | --- | --- | --- | --- |
| Acme | active | 2026-09-01 | true | 129.5 |
| Vandelay | dropped | | false | |
```

`render_config` for that body (`Supplier` has no entry, so it is text):

```json
{
  "columns": {
    "Status": { "type": "select", "options": ["active", "trialling", "dropped"] },
    "Renewal": { "type": "date" },
    "Active": { "type": "checkbox" },
    "Cost": { "type": "number" }
  }
}
```

## Column types and cell validity

| Type | Config entry | Valid cells |
| --- | --- | --- |
| text | none, or `{ "type": "text" }` | Any text. Blank = no value. |
| number | `{ "type": "number" }` | Plain numerals like `4200` or `-1.5` — no thousands separators, no units. Blank = no value. |
| checkbox | `{ "type": "checkbox" }` | `true` or `false`. Blank reads as false and canonicalizes to the literal `false`. |
| date | `{ "type": "date" }` | Exactly `YYYY-MM-DD` (e.g. `2026-07-10`), no time component. Blank = no value. |
| select | `{ "type": "select", "options": ["a", "b"] }` | Exactly one of the declared options, or blank. |
| formula | `{ "type": "formula", "expr": "..." }` | Computed server-side; any hand-written value is rejected with the corrected bytes. See Formulas. |

## Header rules

- Every header cell is non-empty, and headers are unique.
- Config addresses columns by exact header text: every key of `columns` and
  every `column` named in `sort` / `filters` must match a header exactly.
- Formulas address columns by slugified header: lowercase, runs of
  non-alphanumeric characters become `_`, leading/trailing `_` trimmed —
  `Unit Price` is referenced as `unit_price`. In a table with formula
  columns, no two headers may share a slug.

## Formulas

The expression language is numbers, references to other columns by slugified
header, the operators `+ - * /`, and parentheses for grouping. Nothing else:
no comparisons, logic, strings, or function calls; no time-dependent
functions (`now()`, `today()`); no references to other formula columns
(inline that expression instead); every referenced column must have type
`number`.

Values are materialized into the markdown: the server computes each row's
value and writes it into the cell, rounded to 2 decimal places. When a
referenced cell is blank (or a divisor is zero), the formula cell for that
row materializes as blank.

Edit inputs, not outputs. Formula cells cannot hold hand-written values —
the serializer recomputes every formula cell on every write, so a formula
cell that differs from the computed value (stale, wrong, or blank) makes the
body non-canonical and the write is rejected with the corrected bytes in
`canonical_body`. Leave formula cells blank on a first write and resend the
returned `canonical_body`; to change what a formula shows, edit its input
cells or its `expr`.

Canonical example — `2.33` and `1.8` are the real computed values (9800/4200
and 5580/3100), and the Print row's ROAS is blank because its Spend is
blank:

```markdown
| Channel | Spend | Revenue | ROAS |
| --- | --- | --- | --- |
| Google Ads | 4200 | 9800 | 2.33 |
| Meta | 3100 | 5580 | 1.8 |
| Print | | 500 | |
```

```json
{
  "columns": {
    "Spend": { "type": "number" },
    "Revenue": { "type": "number" },
    "ROAS": { "type": "formula", "expr": "revenue / spend" }
  }
}
```

## Common rejections

| Mistake | Error code | Fix |
| --- | --- | --- |
| No pipe table in the body | `no_table` | Send exactly one table: header row, `---` separator row, data rows. |
| More than one table | `multiple_tables` | Split extra tables into separate documents. |
| Prose before or after the table | `prose_not_allowed` | Move it into the document description field. |
| Empty header cell | `empty_header` | Give every column a non-empty header. |
| Same header on two columns | `duplicate_header` | Rename one of the columns. |
| Config names a column that is not a header | `unknown_column` | `columns` keys and `sort` / `filters` columns must match header text exactly. |
| Non-number in a number cell (`abc`) | `invalid_number` | Plain numerals like `4200` or `-1.5`, or leave the cell blank. |
| Anything but true/false in a checkbox cell (`yes`) | `invalid_checkbox` | Use `true` or `false`; blank reads as false. |
| Non-date in a date cell (`July 4`) | `invalid_date` | Exactly `YYYY-MM-DD`, or leave the cell blank. |
| Select value outside the declared options (`paused`) | `unknown_select_option` | Use one of the declared options, or leave the cell blank. |
| Formula references a formula column | `formula_chain_unsupported` | Inline the other expression instead. |
| `now()` or `today()` in a formula | `time_dependent_unsupported` | Materialized values would go stale — use only columns, numbers, `+ - * /`. |
| Formula references an identifier that matches no column | `unknown_column_ref` | Reference columns by slugified header (`Unit Price` as `unit_price`). |
| Formula references a non-number column | `nonnumeric_formula_operand` | Referenced columns must have type `number`. |
| Comparison, logic, string, function call, or unparseable expression | `invalid_expr` | Only numbers, column references, and `+ - * /`. |
| Two headers slugify identically (`Unit Price`, `unit-price`) in a formula table | `slug_collision` | Rename one of the columns. |

## Editing rules

- Make minimal diffs: change only the cells being edited, and resend the
  full body with `Tatara:write_document`.
- Editing a cell rewrites the text between that cell's pipes. Adding a row
  adds one line. Adding, removing, or renaming a column touches the header
  row, the separator row, and every data row — and when that column is typed
  or referenced by config, resend `render_format: "table"` with the full
  updated `render_config` in the same call.
- On gate rejection with `canonical_body`, your change is already in those
  bytes, canonicalized — verify it landed, then resend them unmodified. On
  rejection with structured errors, fix the reported cells. Never retry the
  same body blind.

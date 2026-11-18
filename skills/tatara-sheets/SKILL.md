---
name: tatara-sheets
description: Builds, reads, and changes Tatara workbooks (file_type sheet) - tables with named, typed columns, one formula per computed column, and named cells, all with live formulas. Use when creating or updating a budget, model, quote, forecast, or tracker with totals in Tatara, when totalling, counting, or looking up across rows of a workbook, or when a write_sheet call is rejected. Covers the describe, read, calculate, write loop, the session and op_id, the ops grammar, formula references, where filters, recording the call a table's rows came from, working alongside other agents, and how to recover from each refusal.
license: MIT
compatibility: Requires the Tatara MCP server connected
metadata:
  author: tatara-app
---

# Tatara workbooks

A workbook (`file_type: sheet`) holds tables and named cells. A table has
named, typed columns. A column is either stored (you write its values) or
computed (one formula fills every row). A named cell holds one value or one
formula. People see a workbook as a read-only spreadsheet in the Tatara app;
agents read and change it with four tools.

| Tool | Use it to |
| --- | --- |
| `Tatara:describe_sheet` | Get the workbook's outline and a `session`. Always first. |
| `Tatara:read_sheet` | Read rows of one table, named cells, or the change history. |
| `Tatara:calculate_sheet` | Evaluate formulas against the workbook. Nothing is stored. |
| `Tatara:write_sheet` | Create a workbook, or apply a batch of changes to one. |

A workbook's content is written only with `Tatara:write_sheet`.
`Tatara:write_document` and `Tatara:update_document` refuse it, and
`Tatara:get_document` returns its metadata without content. Search, move,
delete, and restore work as for any document.

## Start with the outline

Call `Tatara:describe_sheet` with the workbook's `id` or `path`. It returns
the purpose and notes, every table (its description, row count, when it
last changed and who changed it (`changed`), the call that produced its rows
if one is recorded (`source`), and each column's type, unit, formula, and
range of values), the named cells with their values and formulas, any error
cells, recent changes, and a `session`. The outline is the same size however
many rows the workbook has.

Pass that `session` to every later call on the same workbook. If a call says
the session is not known or has expired, call `Tatara:describe_sheet` again
for a new one and read what you need again.

Values in the outline are for finding your way. Before you rely on a computed
value, read it with `Tatara:read_sheet` or `Tatara:calculate_sheet`.

## Reading

`Tatara:read_sheet` takes exactly one of:

- `table`: rows of that table. Choose `columns`, filter with `where`, and
  page with `limit` (default 50, at most 200) and `offset`. Each row comes
  back as one array led by its row id, such as `r17`. The read also returns
  the table's `description` and its full `source`, if one is recorded.
- `names`: a list of named cells, or `"all"`.
- `history`: `true` for the recent batches, or one batch's version number
  for its changes.

Values are shown as they are compared: numbers to 15 significant digits,
dates as `"YYYY-MM-DD"`, errors as `{"error":"#N/A"}`, and blank cells as
`null`.

`where` is `{"column": {"operator": value}}`. Every condition must hold.

| Column type | Operators | Value |
| --- | --- | --- |
| `number` | `eq` `ne` `gt` `gte` `lt` `lte` | a number |
| `date` | `eq` `ne` `gt` `gte` `lt` `lte` | `"YYYY-MM-DD"` |
| `text` | `eq` `ne` `contains`, all ignoring case | a string |
| `boolean` | `eq` `ne` | `true` or `false` |

`{"eq": null}` matches a blank cell and `{"ne": null}` a filled one. A cell
holding an error matches no `eq` and no `gt` to `lte`.

To total, count, average, or look up, call `Tatara:calculate_sheet` with 1
to 20 formulas, for example `["=SUM(line_items[amount])"]`. Do not read rows
and add them up yourself.

## Writing

`Tatara:write_sheet` takes `op_id` and `ops`, and either:

- `create`: `{title, folder, description, tags}` (1 to 5 tags, as for any
  document) to make a new workbook. Its `ops` usually start with
  `add_table`. The result carries the new workbook's `id`, `url`, and
  `session`.
- `id` or `path`, with `session`, to change an existing workbook.

A batch is 1 to 100 operations that set or remove at most 5,000 cells. All
of it is written, or none of it.

Give each batch a new `op_id`. If a call times out, or says the calculation
could not be run, send the same batch again with the same `op_id`: it is
never applied twice. Never reuse an `op_id` for a different batch.

### Operations

One line an operation:

```
{"op":"add_table","name":"line_items","description":"…","columns":[{"name":"line_id","type":"text"},{"name":"qty","type":"number"},{"name":"unit_cost","type":"number","unit":"AUD"},{"name":"amount","type":"number","unit":"AUD","formula":"=[qty]*[unit_cost]"}],"rows":[["L-1",2,195]]}   "rows" is optional: one array a row, its values in the order of the table's stored columns (those with no formula); so is "source", as set_table takes it
{"op":"add_rows","table":"line_items","columns":["line_id","qty"],"rows":[["L-9",2]]}   computed columns fill themselves
{"op":"update","table":"projects","row":"r17","set":{"system_kw":300}}   one row by id
{"op":"update","table":"projects","where":{"project_id":{"eq":"P-014"}},"set":{…}}   one row by content; refused unless exactly one row matches
{"op":"delete_rows","table":"line_items","rows":["r9"]}
{"op":"add_column","table":"line_items","name":"tax","type":"number","formula":"=[amount]*gst_rate"}   omit "formula" for a stored column
{"op":"set_column","table":"line_items","column":"tax","formula":"=[amount]*0.2"}   a computed column's formula; any column's unit or description
{"op":"delete_column","table":"line_items","column":"tax"}
{"op":"set_table","table":"alarms","description":"…","source":{"connector":"telemetry","tool":"query","args":{"dataset":"alarm_counts","day":"2026-11-17"},"note":"day is the latest complete day when the call is made."}}   a table's description, or the call that produced its rows: connector, tool and arguments as you made it, and, if an argument moves with time, a note saying which and what it is; either or both; null clears one
{"op":"delete_table","table":"line_items"}
{"op":"set_name","name":"gst_rate","value":0.1,"unit":"ratio","note":"…"}   or "formula":"=SUM(line_items[amount])"
{"op":"delete_name","name":"gst_rate"}
{"op":"set_purpose","purpose":"…","notes":["…"]}   purpose at most 500 characters; at most 10 notes of 300 characters each
```

Column types are `text`, `number`, `date`, and `boolean`. Send a date as
`"YYYY-MM-DD"`, from 1900-03-01 on.

Tables, columns, and named cells are named in lowercase letters, digits,
and underscores, start with a letter, and are at most 31 characters, such
as `gross_margin`. Tables and named cells share one set of names. A name
that looks like a cell address, such as `a1` or `r2c3`, is refused.

### Formulas

- A formula starts with `=`.
- `[qty]` is this row's value. It works only in a computed column.
- `line_items[amount]` is a whole column.
- `gst_rate` is a named cell.
- There are no cell addresses (`A1`) and no sheet references, so `INDIRECT`
  and `OFFSET` are not available.
- `TODAY`, `NOW`, `RAND`, `RANDBETWEEN`, and `RANDARRAY` are not available.
  For today's date, keep a date named cell such as `as_of` and set it
  yourself.
- `LET`, `LAMBDA`, `CELL`, `INFO`, `FORMULATEXT`, `SHEET`, and `SHEETS` are
  not available.
- A formula gives one value. A named cell's formula has no row, so it uses
  whole columns inside a function that returns one value, such as `SUM` or
  `INDEX`.
- `SUMIFS`, `COUNTIFS`, `AVERAGEIFS`, `MAXIFS`, and `MINIFS` take all their
  ranges from one table.

### Good practice

- Anything calculated is a formula: a computed column (`=[qty]*[unit_cost]`)
  or a named cell (`=SUM(line_items[amount])`). Never type in a number you
  worked out.
- Put rates and assumptions in named cells, each with a `unit` and a `note`.
- Wrap money made by multiplying or dividing in `ROUND(x, 2)`.
- Look up rows with an exact match: `XLOOKUP`, or `INDEX` with
  `MATCH(…, 0)`. An approximate lookup warns. A key in a stored number
  column with no blanks is the fastest to look up.
- Say what the workbook is for with `set_purpose`, so the next agent knows.

## After every write

Read the response. It gives the recalculated value of everything you wrote,
how many other cells changed, anything that is now an error and why,
warnings, and how long the recalculation took. Fix an error at its cause,
not by typing a value over it.

A recalculation over 1 second warns and names the costliest formulas; one
over 5 seconds is refused. A workbook holds at most 500,000 cells.

## Working alongside others

Other agents and people may use the same workbook. Tatara tracks what each
session has read:

- Read a row or named cell before you change or delete it.
- If anything you read has changed since you read it, the batch is refused.
  The refusal says what changed, who changed it, and when, and shows the
  current values or the row ids. Do what it says (check your batch against
  the values, read the listed rows, or calculate again), then resend with a
  new `op_id`.

## Rows that came from another system

When you load a table's rows from another system, such as a query through
one of your connectors, record the call that produced them, so the next
agent can run it again:

```
{"op":"set_table","table":"alarms","source":{"connector":"telemetry","tool":"query","args":{"dataset":"alarm_counts","day":"2026-11-17"},"note":"\"day\" moves with time: it is the latest complete day when the call is made."}}
```

- Send the call as you made it: `connector`, `tool`, and `args`. `note` is
  optional.
- If an argument moves with time, such as a day or a month, say so in the
  note and say what it is. On a refresh, set it to what the note says, not
  to the stored value.
- After a refresh with a changed call, send `set_table` again with the new
  call.
- `set_table` changes a table's `description`, its `source`, or both, and
  `null` clears one. It changes no cell. `add_table` takes the same
  `source`.
- `connector` is at most 80 characters, `tool` 120, and `note` 300; `args`
  is at most 2,000 bytes as JSON.

`Tatara:describe_sheet` shows each table's source, with who recorded it and
when (`declared`), and when the table last changed (`changed`). When the
outline has no room left for a source, it shows `"source":{"more":true}`:
`Tatara:read_sheet` on that table returns it in full.

## Common rejections

Every refusal ends "Nothing was written." Read the sentence and apply the
fix; never resend the same batch unchanged with a new `op_id`.

| The refusal says | Fix |
| --- | --- |
| "`amount` is computed by `=[qty]*[unit_cost]`…" | You sent a value for a computed column. Leave it out, or change its inputs or its formula (`set_column`). |
| "…holds stored values, so a cell cannot hold a formula" or "…so it cannot take a formula" | Add a computed column (and delete the stored one), or use a named cell. |
| "…you read have changed since you read them" or "Rows have been added to…" | Another session changed what you read. Follow the refusal, then resend with a new `op_id`. |
| "…a row this session has not read" or "…a named cell this session has not read" | Read it with `Tatara:read_sheet`, then resend. |
| "`where` matched no row" or "`where` matched 3 rows" | An `update` by `where` must match exactly one row. Name the row by its id. |
| "This session is not known or has expired" | Call `Tatara:describe_sheet` for a new session, read what you need again, then resend. |
| "`TODAY` is not available" (or another function) | Use a named cell or a computed column instead. |
| "`SUMIFS` needs all its ranges from one table" | Use whole columns of one table, written as `table[column]`. |
| "`op_id` … was already used for a different batch" | Use a new `op_id`. |
| "The calculation could not be run" | Send the same batch again with the same `op_id`. |
| "Recalculating after this batch took more than 5 seconds" | Simplify or remove the costliest formulas it names. |

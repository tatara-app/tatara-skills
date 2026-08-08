---
name: tatara
description: Works with a company's Tatara brain over the Tatara MCP server - finding, reading, creating, organizing, and recovering governed documents and folders. Use when any task involves Tatara or its MCP tools - writing or updating documents, creating or moving folders, searching or listing company knowledge, tagging with the taxonomy, choosing a render format (kanban board, task tracker, data table, base view, mind map), or restoring deleted items. Covers the write workflow (search first, resolve the folder, then write), slug and wikilink behavior, universal keys, render_format selection, and how to recover when a write is rejected by a validation gate.
license: MIT
compatibility: Requires the Tatara MCP server connected
metadata:
  author: tatara-app
---

# The Tatara brain

## The model

- A brain is a company's governed knowledge store: folders holding markdown
  documents (plus static HTML documents). Agents and humans read and write
  the same documents over the Tatara MCP server.
- Four universal keys live on every document — `title`, `type`,
  `description`, `tags` — set through explicit tool inputs, never as a YAML
  frontmatter block inside `body`. Universal keys placed in the optional
  `frontmatter` object are ignored (the explicit inputs are authoritative);
  `frontmatter` is for custom keys only. `source` is stamped from your auth
  token — never send it.
- Slugs are derived, never supplied: a document's slug and path come from its
  title, a folder's from its name. Retitling a document or renaming a folder
  re-derives its slug and path, and everything affected — descendant paths,
  inbound wikilinks — updates automatically.
- Documents reference each other with `[[wikilinks]]`. `[[Note Title]]` (no
  `/`) targets a document by its title, case-insensitively — if several
  documents share the title, the shallowest path wins, then alphabetical.
  `[[path/to/doc]]` (contains `/`) targets by exact full path.
  `[[target|display text]]` renders the display text. `Tatara:get_document`
  returns each document's outbound `links` and inbound `backlinks` — follow
  them with further calls to traverse the knowledge graph.

## Tool map

**Read / search**

| Tool | Purpose |
| --- | --- |
| `Tatara:search_documents` | Full-text search across the brain — ranked results with snippets. Optional `type`, `folder`, and `tags` filters compose with the query. |
| `Tatara:list_documents` | Browse/enumerate when you have no search terms ("what is in this folder?", "what changed since …?"). Lightweight rows only; scope with `folder`, filter with `updated_since`, paginate with `cursor`. |
| `Tatara:get_document` | Read one document by `id` or `path`: body, title, type, tags, custom metadata, links, and backlinks. |
| `Tatara:get_taxonomy` | The brain's current shape: the real folders (write destinations), the document types in use, the tag registry, and the render-format catalog with canonical examples. Call once at the start of a session that may write; cache it. |

**Write**

| Tool | Purpose |
| --- | --- |
| `Tatara:write_document` | Create or update a markdown document. Omit `target_document_id` to create (`type`, `folder`, `title`, `description`, `body`, `tags` all required); pass it to update as a partial patch — only the fields you send change. |
| `Tatara:update_document` | Line edits: ordered string replacements (`old_string` → `new_string`) applied to the body only — title, type, tags, folder, and frontmatter are preserved. |
| `Tatara:create_html_document` | Create a self-contained, static HTML report, dashboard, or visual summary (inline CSS, images as data: URIs, no JavaScript). Use when the visual layout is the point; for prose, write markdown. Create-only — revising an HTML document means writing a replacement. |

**Organize**

| Tool | Purpose |
| --- | --- |
| `Tatara:create_folder` | Create exactly one folder: `name` plus optional `parent` (omit for top level). Reuse an existing folder wherever one fits; the slug is derived from the name. |
| `Tatara:move_document` | Move a document to a different folder, or to the top level when `target_folder` is omitted. |
| `Tatara:move_folder` | Move a folder — and everything inside it — under a new parent, or to the top level; contained paths update automatically. |
| `Tatara:rename_folder` | Rename a folder; its slug and path re-derive from the new name and the paths of everything inside update to match. |

**Recover**

| Tool | Purpose |
| --- | --- |
| `Tatara:delete_document` | Soft-delete a document (recoverable with `Tatara:restore_document`). Rejected while any agent still references the document — the rejection names those agents. |
| `Tatara:restore_document` | Un-delete a document — back to its original folder if that still exists, else the top level. |
| `Tatara:delete_folder` | Soft-delete a folder. An empty folder deletes immediately; a non-empty one is rejected with an inventory — confirm with the user, then retry with `cascade: true` to soft-delete the subtree. |
| `Tatara:restore_folder` | Un-delete a folder and everything that was deleted inside it, in one call. |

## Standing rules

1. **Search before creating.** Run `Tatara:search_documents` before proposing
   a new document — duplicates are common. Update the existing document
   rather than writing a twin.
2. **`Tatara:get_taxonomy` before filing or tagging.** Folders and tags are
   the brain's real, current shape, not a fixed schema. Tag from the
   registry: exactly one tier-1 (area), up to two tier-2 (topic), up to two
   tier-3 (goal) — 1–5 tags total. Coin a new short kebab-case tag only when
   nothing in the registry fits; it is accepted and flagged for curation.
3. **Folders resolve by path first.** A folder reference is a slug or a full
   path. An exact path always wins; a bare slug works only while it is
   unique — slugs are unique only among siblings, so a slug naming several
   folders is rejected with the candidate paths listed. Re-send the full
   path of the one you mean. An unknown folder is rejected, never
   auto-created — create one deliberately with `Tatara:create_folder`.
4. **Updates are partial.** `Tatara:write_document` with
   `target_document_id` changes only the fields you send — omitted fields,
   including `body` and `render_format`, are preserved exactly. To move as
   part of a wider update, send `target_document_id` and `folder`; for a
   pure move — and the only route to the top level — use
   `Tatara:move_document`. To rename, send `title`; to retag, `tags`.
   Never resend a body to change metadata.
5. **Prefer `Tatara:update_document` for edits within a plain body.** Each
   `old_string` must appear in the body and be unique (or pass
   `replace_all`); on any no-match or ambiguous match the whole call is
   rejected and nothing is written. For documents with a `render_format`,
   follow the sibling skill's editing rules instead.
6. **`body` is the markdown body only** — no YAML frontmatter block, and no
   H1 repeating the title (the title lives in the `title` field; open the
   body on content, using `##` and below for headings). `description` is one
   plain sentence stating what the document is; it powers listing and
   search.

## Format catalog

A markdown document may carry a `render_format`, giving it a rich
interactive view over a canonical body grammar. Set it when creating, or
send it on an update — converting an existing document, or changing a
table's config — and the gate validates the document's effective body
against the format. `render_config` declares table's typed columns and
always requires `render_format`. On update an omitted `render_format` is
preserved — moved or edited documents keep their format.

| `render_format` | Pick when the document is | Grammar |
| --- | --- | --- |
| `kanban` | a workflow board — columns of cards | sibling skill `tatara-kanban` |
| `tracker` | a task checklist with tags, priority, and dates | sibling skill `tatara-tracker` |
| `table` | typed records in a single grid | sibling skill `tatara-table` |
| `base` | a saved view / index over the brain's other documents | sibling skill `tatara-base` |
| `mindmap` | branching structure — topics radiating from the title | sibling skill `tatara-mindmap` |
| omitted | plain prose or notes — the default | — |

Each format's body must parse and be byte-canonical; load the matching
sibling skill before authoring or editing one.

## Gate recovery

Every rejected write returns an instructive payload — read it and apply the
stated fix; never retry the same payload blind.

- `{ status: "rejected", reason }` states exactly what to change (unknown
  folder, missing target, document cap, …). Ambiguous folder rejections add
  `candidates`; HTML rejections add per-rule `issues`.
- Format-gated writes add structured `errors` (each with a fix instruction
  and, where it applies, a line number) when the body does not parse — fix
  the reported lines.
- A format body that parses but is not byte-canonical is rejected with the
  exact corrected bytes in `canonical_body` — your change is already in
  them, canonicalized. Verify it landed, then resend them unmodified.

## Recovery of deletions

Restores report where the item landed: `Tatara:restore_document` returns
the document's (possibly relocated) path. `Tatara:restore_folder` is
addressed by the slug or path the folder had before deletion — a deleted
folder is no longer in the taxonomy — and is rejected if a live folder now
occupies that path or the parent folder is still deleted.

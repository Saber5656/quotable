# ADR-002: SQLite is canonical; Markdown is a one-way export

Status: Accepted (2026-07-08, confirmed with product owner)

## Context

Highlights need durable storage the user owns, fast full-text search, reliable
deduplication across repeated imports, and an Obsidian-friendly output. Candidate
models: Markdown files as the source of truth, bidirectional SQLite⇄Markdown sync, or
SQLite canonical with one-way Markdown export.

## Decision

A single SQLite database (`<data-dir>/quotable.db`) is the only source of truth.
`quotable export markdown` renders it to Markdown files (one per book) in a
user-chosen directory. Exported files are regenerated deterministically on each export;
edits made to exported Markdown are **not** read back and will be overwritten.

## Consequences

- Dedup (`UNIQUE (source, dedup_key)`), FTS5 search, and schema migrations stay
  straightforward; import idempotency is enforceable by the database.
- Users must treat exported Markdown as read-only build artifacts. This is stated in
  the export command help, the README, and the exported frontmatter
  (`quotable_book_id` marks generated files).
- Obsidian users get vault integration by pointing `--out` at a vault folder.
- Bidirectional sync (conflict resolution, file watching) is deferred; if ever built,
  it becomes a v2+ design with its own ADR.

## Alternatives rejected

- **Markdown canonical**: human-editable, but dedup/search/consistency across
  re-imports would depend on parsing user-edited files — fragile and much larger to
  implement correctly.
- **Bidirectional sync**: best UX on paper; conflict-resolution design is too heavy
  for v1 and multiplies failure modes.

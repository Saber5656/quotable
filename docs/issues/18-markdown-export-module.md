# Title

Markdown export module: rendering and path safety

## Summary

Implement `src/core/export/markdown.ts` (deterministic per-book Markdown rendering
with the fixed v1 template) and `src/core/export/files.ts` (filename sanitization and
output-path containment). CLI wiring is issue 19.

## Context

Export is one-way from the canonical SQLite DB (ADR-002). Template, naming, and path
safety are DESIGN.md §9; path safety is security boundary B5 (§11.1).

## Scope

- The two modules + unit tests. No CLI.

## Detailed Requirements

1. `files.ts`:
   - `sanitizeFilename(title: string): string` — NFKC-normalize; strip control chars
     (U+0000–U+001F, U+007F); replace `/ \ : * ? " < > |` and leading `.` with `-`;
     collapse whitespace to single spaces, trim; cap 100 code points; empty result →
     `untitled`.
   - `bookFileName(title, bookId)` → `<sanitized>--<bookId>.md`.
   - `resolveOutputPath(outDir, fileName, dataDir)`:
     - `outDir` resolved absolute (expand `~`); created `mkdir -p` if missing.
     - Final joined path must satisfy
       `path.resolve(outDir, fileName).startsWith(path.resolve(outDir) + path.sep)`
       → else `AppError('PATH_ESCAPE')` (defense in depth; sanitize should prevent).
     - If resolved `outDir` equals or contains the quotable data dir (or vice versa)
       → `AppError('INVALID_OUT_DIR')`.
2. `markdown.ts`:
   - `renderBookMarkdown(book, highlights, opts: { now: string }): string` — exact
     template of DESIGN.md §9:
     - YAML frontmatter: `title`, `author` (omit line when null), `source:
       apple_books`, `quotable_book_id`, `exported_at`, `highlights` (count). String
       values use double-quote style with a full escaper (single function, exported
       for tests): `\` → `\\`, `"` → `\"`, CR/LF → single space, other control
       characters (U+0000–U+001F, U+007F) stripped.
     - Body: `# {title}` then `## Highlights`; per highlight (input order preserved):
       blockquote with every line of `text` prefixed `> `; optional `- Note:` (note
       rendered inline, newlines → single spaces), `- Chapter:`, `- Tags: #a #b`
       (tag names with internal spaces → hyphens for the hashtag form), and a final
       metadata line rendered by these exact rules (`styleName()` from
       `src/core/highlightStyle.ts`, issue 12):
       - date and style present → `- Highlighted: <YYYY-MM-DD> · Style: <name>`
       - date only → `- Highlighted: <YYYY-MM-DD>`
       - style only → `- Style: <name>`
       - both null → omit the line entirely.
     - Deterministic: same inputs → byte-identical output (no locale/timezone
       dependence; dates rendered from the stored ISO strings, UTC).
   - `exportBooks(db, { outDir, dataDir, bookId?, includeArchived?, now }):
     { files: { path: string; bookId: string; highlights: number }[] }` — selects
     books with ≥1 matching highlight, ordered highlights by
     `location_start ASC NULLS LAST, id`, writes files (UTF-8, `\n` endings,
     overwrite existing), returns the manifest. Never deletes anything.
3. Archived highlights excluded unless `includeArchived`.

## Acceptance Criteria

- Snapshot test: JP book with notes/tags/multiline text renders to a golden file;
  emoji and `"` in title escape correctly in YAML.
- Hostile titles (`../../etc/passwd`, `a/b:c*?.md`, 300-char CJK, empty) sanitize to
  contained names; `PATH_ESCAPE` unreachable through public API but unit-tested
  directly.
- `outDir` inside data dir (and data dir inside outDir) rejected.
- Determinism: two runs produce byte-identical files; multiline text always `> `
  prefixed per line.
- Books with zero (non-archived) highlights produce no file.

## Validation

- `npm test` with golden snapshots committed under `test/__snapshots__/`.

## Dependencies

- 06, 07, 12 (for `src/core/highlightStyle.ts`)

## Non-goals

- Custom templates (v2), deletion/pruning of stale files, CLI flags (19).

## Design References

- DESIGN.md §9, §11.1 B5; ADR-002

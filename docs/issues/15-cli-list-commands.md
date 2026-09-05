# Title

CLI commands: `list books` and `list highlights`

## Summary

Implement `quotable list books [--sort title|recent]` and
`quotable list highlights [--book <id>] [--tag <name>] [--favorites] [--archived]
[--limit N] [--offset N]` with table and `--json` output.

## Context

Read surfaces over the repositories (DESIGN.md §8.2). Output conventions from issue
05 apply; repository filters from issue 06.

## Scope

- `src/cli/commands/list.ts` + child-process tests.

## Detailed Requirements

1. `list books`:
   - Columns: `ID` (first 8 chars), `TITLE` (≤50 display chars), `AUTHOR` (≤30),
     `HIGHLIGHTS` (count). `--sort` default `title`.
   - `--json`: `{ items: [{ id, title, author, highlightCount }], total }` (full IDs).
2. `list highlights`:
   - `--book` accepts full ID or ≥6-char unique prefix (`resolveBookId`); ambiguous →
     exit 2 listing candidates on stderr.
   - `--tag` filters via normalized tag name; unknown tag → empty result (not error).
   - `--favorites` → `favorite=1`; `--archived` → only archived (default excludes).
   - `--limit` default 50 max 200, `--offset` default 0; values out of range → exit 2.
   - Table columns: `ID` (8), `BOOK` (≤30), `TEXT` (≤60, single line, newlines →
     `⏎`), `TAGS` (comma-joined), `FAV` (`*` or empty).
   - `--json`: `{ items: HighlightItem[], total, limit, offset }` where
     `HighlightItem` is exactly the issue 06 shape: `{ id, bookId, bookTitle,
     bookAuthor, text, note, chapter, location, style, styleName, favorite,
     archived, tags, highlightedAt, importedAt }`.
3. Empty result: human mode prints `No books found.` / `No highlights found.` to
   stdout, exit 0.
4. No FTS involvement here (that is `search`, issue 16).

## Acceptance Criteria

- All flags and combinations covered by child-process tests against a seeded temp DB
  (seed via repositories, not fixtures).
- JSON shapes exact (snapshot with stable seed data).
- Prefix resolution and ambiguity behavior proven.
- CJK text truncation counts display width naively by code points (documented; no
  east-asian-width dependency in v1).

## Validation

- `npm test`.

## Dependencies

- 05, 06, 07

## Non-goals

- Search (16), interactive pager, color output.

## Design References

- DESIGN.md §8.1, §8.2

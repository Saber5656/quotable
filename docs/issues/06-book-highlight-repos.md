# Title

Book and highlight repositories

## Summary

Implement `src/core/repo/books.ts` and `src/core/repo/highlights.ts`: typed CRUD and
query functions over schema v1 used by the import pipeline, CLI, and API.

## Context

Domain rules (identity, ownership of fields) are DESIGN.md §5; consumers are issues
13, 15, 21, 22. Repositories are plain functions taking a `Database` first argument —
no ORM, no classes.

## Scope

- The two repo modules + unit tests. Tag operations belong to issue 07.

## Detailed Requirements

1. `books.ts`:
   - `upsertBook(db, input: { source: 'apple_books'; sourceAssetId: string;
     title: string; author?: string|null; genre?: string|null;
     language?: string|null }): { book: BookRow; created: boolean; updated: boolean }`
     — identity `(source, source_asset_id)`; updates title/author/genre/language only
     when changed; maintains `created_at`/`updated_at`.
   - `getBook(db, id)`, `listBooks(db, { sort: 'title'|'recent' })` — `recent` =
     most recently updated highlight's `imported_at` desc, books with none last;
     each row includes `highlightCount` (non-archived).
   - ID prefix resolution: `resolveBookId(db, idOrPrefix)` — exact match wins; else
     unique prefix (>= 6 chars) matches; 0 matches → `AppError('NOT_FOUND')`,
     >1 → `AppError('AMBIGUOUS_ID', details: candidates)`.
2. `highlights.ts`:
   - Input types (exact):
     ```ts
     interface HighlightInsertInput { bookId: string; source: 'apple_books';
       sourceUuid: string | null; dedupKey: string; text: string;
       note: string | null; chapter: string | null; location: string | null;
       locationStart: number | null; style: number | null;
       highlightedAt: string | null; sourceModifiedAt: string | null; }
     // id, imported_at, updated_at generated inside; favorite/archived default 0
     interface HighlightSourceUpdateInput { text: string; note: string | null;
       chapter: string | null; location: string | null;
       locationStart: number | null; style: number | null;
       sourceModifiedAt: string | null; }
     ```
   - `insertHighlight(db, input: HighlightInsertInput)` /
     `updateSourceFields(db, id, input: HighlightSourceUpdateInput)` /
     `setArchived(db, id, archived: boolean)` — field sets exactly per DESIGN.md §5
     (import never touches `favorite`; `updateSourceFields` bumps `updated_at`).
   - `findBySourceKey(db, source, dedupKey)`.
   - `listHighlights(db, { bookId?, tagNorm?, favorite?, archived?, limit, offset })`
     → `{ items: HighlightItem[], total, limit, offset }` where
     ```ts
     interface HighlightItem { id: string; bookId: string; bookTitle: string;
       bookAuthor: string | null; text: string; note: string | null;
       chapter: string | null; location: string | null; style: number | null;
       styleName: string | null; favorite: boolean; archived: boolean;
       tags: string[]; highlightedAt: string | null; importedAt: string; }
     ```
     (`styleName` derived via `src/core/highlightStyle.ts`, issue 12; until that
     lands, emit `null` — no hard dependency). Tag joins must not duplicate items
     (aggregate tags per highlight). Ordering
     `book_title ASC, location_start ASC NULLS LAST, id ASC`; default excludes
     archived. Pagination validation identical to issue 04 (`limit` int 1–200
     default 50, `offset` int ≥ 0 default 0, else `AppError('VALIDATION')`).
   - `getHighlight(db, id)` (embeds tags + book fields) and
     `resolveHighlightId(db, idOrPrefix)` (same prefix rules as books).
   - `setFavorite(db, id, favorite: boolean)`.
3. All timestamps via `nowIso()`; all new IDs via `newId()`.
4. Every function throws `AppError('NOT_FOUND')` for missing IDs (no silent nulls on
   mutation paths); read paths may return `undefined` where typed so.

## Acceptance Criteria

- Upsert semantics proven: same asset twice → one row, `created:false`; changed title
  → `updated:true` and `updated_at` bumped.
- `listHighlights` filter matrix tested (bookId, favorite, tag, archived,
  pagination totals); a highlight with 3 tags appears exactly once; invalid
  limit/offset → `VALIDATION`.
- Prefix resolution: exact, unique-prefix, ambiguous, too-short (<6 chars → treated
  as not-found unless exact) cases tested for both entities.
- `updateSourceFields` cannot change `favorite` (type-level and test).

## Validation

- `npm test` — in-memory DB seeded via issue 03 helpers.

## Dependencies

- 03

## Non-goals

- Tag CRUD (07), FTS search (04), import decisions (13).

## Design References

- DESIGN.md §4, §5, §8.2

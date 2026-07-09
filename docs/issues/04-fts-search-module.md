# Title

FTS5 search query module

## Summary

Implement `src/core/search.ts`: one shared search function over highlights with FTS5
phrase matching, a LIKE fallback for short queries, filters, pagination, snippets, and
hardened input escaping. CLI (issue 16) and API (issue 21) call this module.

## Context

Search behavior is specified in DESIGN.md §7; injection resistance is a security
requirement (§11.4). CJK substring search relies on the trigram tokenizer from
migration 001.

## Scope

- `src/core/search.ts` + unit tests. No CLI/API wiring.

## Detailed Requirements

1. Export:
   ```ts
   interface SearchFilters { bookId?: string; tag?: string; favorite?: boolean;
     archived?: boolean; }        // archived defaults to false (excluded)
   interface SearchParams { query: string; filters?: SearchFilters;
     limit?: number; offset?: number; }   // limit default 50, max 200
   interface SearchHit { highlightId: string; bookId: string; bookTitle: string;
     bookAuthor: string | null; text: string; note: string | null; snippet: string;
     favorite: boolean; tags: string[]; location: string | null; }
   interface SearchResult { items: SearchHit[]; total: number; limit: number;
     offset: number; }
   search(db: Database, params: SearchParams): SearchResult
   ```
   Pagination validation: `limit` must be an integer in 1–200 and `offset` an
   integer ≥ 0 (defaults 50 / 0); anything else → `AppError('VALIDATION')`. Never
   pass negative values to SQL (`LIMIT -1` would disable the cap).
2. Query routing: count Unicode code points of `query.trim()`.
   - `>= 3` → FTS path: `highlights_fts MATCH ?` where the parameter is the trimmed
     query wrapped as one double-quoted phrase with internal `"` doubled
     (`escapeFtsPhrase(q)` exported for testing). User input must never reach FTS
     query syntax (no `*`, `OR`, `NEAR`, column filters).
   - `1..2` → LIKE path over `text` and `note` with `%`, `_`, `\` escaped in the bound
     parameter and `ESCAPE '\'`.
   - Empty/whitespace query → `AppError('EMPTY_QUERY')`.
3. Filters compose with either path: `bookId` (equality), `tag` (join via
   `highlight_tags`/`tags.name_norm` of the normalized input), `favorite = 1`,
   `archived` (default `WHERE archived = 0`; `archived: true` shows only archived).
4. All SQL uses bound parameters; no string interpolation of user input.
5. Ordering: `bookTitle ASC, location_start ASC NULLS LAST, highlights.id ASC`.
6. Snippet: FTS path uses `snippet(highlights_fts, 0, '«', '»', '…', 20)`; if the
   returned string contains no `«` marker (match was in `note`), query
   `snippet(highlights_fts, 1, ...)` and use that. LIKE path returns first 160
   code points of `text` with `«»` inserted around the first match (from `text`, or
   from `note` when only `note` matched). Markers `«»` are plain strings for
   consumers to render safely.
7. `total` computed with a matching COUNT query (same WHERE).

## Acceptance Criteria

- CJK ("読書感想" — 4 code points → FTS path) and ASCII queries return correct rows
  via FTS; 2-char CJK query ("読書" → LIKE path) still matches; a single-emoji query
  (1–2 code points → LIKE path) matches a highlight containing that emoji.
- Invalid pagination (`limit: 0`, `limit: 999`, `offset: -1`, non-integers) →
  `AppError('VALIDATION')`.
- Hostile inputs return zero-or-correct results without error and without acting as
  operators: `"`, `""`, `*`, `(`, `)`, `NEAR/2`, `text:foo`, `%`, `_`, `\`,
  `'; DROP TABLE highlights; --`.
- Filters and pagination proven by tests (favorite+tag+book combined case included).
- Archived highlights absent from default results.

## Validation

- `npm test` — table-driven tests seeded via issue 03's schema with fixture rows
  (JP novel text, EN text, emoji, 10k-char text).

## Dependencies

- 03

## Non-goals

- Ranking/relevance tuning, prefix search, multi-term boolean queries (v2).

## Design References

- DESIGN.md §7, §11.4

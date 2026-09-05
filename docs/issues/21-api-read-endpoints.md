# Title

API read endpoints: books, highlights, search, tags

## Summary

Implement `GET /api/books`, `GET /api/books/:id`, `GET /api/highlights`,
`GET /api/tags` on the hardened server, with JSON-schema validation and the shared
list envelope.

## Context

DESIGN.md §10.2 table; data comes from repositories (06/07) and `core/search.ts`
(04 — the `q` parameter routes `GET /api/highlights` through it).

## Scope

- `src/server/routes/books.ts`, `highlights.ts` (read part), `tags.ts` + inject
  tests. Mutations are issue 22.

## Detailed Requirements

1. `GET /api/books?sort=title|recent` (default `title`; other values → 400):
   `{ items: [{ id, title, author, genre, language, highlightCount }], total }`.
2. `GET /api/books/:id`: full ID only (no prefixes over HTTP); missing → 404
   `NOT_FOUND`. Shape: book fields + `highlightCount`.
3. `GET /api/highlights` query params (all optional, unknown → 400):
   `bookId`, `tag`, `favorite` (bool), `archived` (bool, default false), `q`
   (string), `limit` (1–200, default 50), `offset` (≥0):
   - without `q`: repository listing (issue 06) — items are `HighlightItem`
     (exactly the issue 06 shape, camelCase, includes `styleName`).
   - with `q`: delegate to `core/search.ts` with filters; items are `SearchHit`
     (exactly the issue 04 shape — includes `bookAuthor` and `snippet`).
   - Response envelope `{ items, total, limit, offset, mode: 'list' | 'search' }`.
   `/api/books` and `/api/tags` are unpaginated in v1 (DESIGN.md §10.2): shapes are
   exactly `{ items, total }` and `{ items }` respectively.
4. `GET /api/tags`: `{ items: [{ id, name, count }] }` (issue 07 ordering).
5. Every route declares Fastify response schemas too (guards against accidental
   field leakage).
6. All routes side-effect free; rely on issue 20's global hooks (no per-route
   security code).

## Acceptance Criteria

- inject-tests over a seeded DB cover: sorting, 404, each filter, `q` switching to
  search mode (snippet present), pagination totals, bad params (unknown field,
  `limit=999`, `favorite=maybe`) → 400 envelope.
- CJK search via API returns the seeded row.
- Response schema strips nothing needed and leaks nothing extra (snapshot of one
  item per mode).
- Security headers present on these routes (inherited; one assertion).

## Validation

- `npm test`.

## Dependencies

- 04, 06, 07, 20

## Non-goals

- Mutations and imports (22), UI consumption (24/25).

## Design References

- DESIGN.md §7, §10.2

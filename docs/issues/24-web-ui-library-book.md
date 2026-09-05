# Title

Web UI: library and book detail views

## Summary

Implement the Library view (`/`) with book cards, sort toggle, and the
import-from-Apple-Books button, and the Book detail view (`/books/:id`) with the
highlight list, favorite toggles, inline tag editing, and the archived toggle.

## Context

DESIGN.md §10.3 route table; consumes issues 21/22 endpoints via the issue 23 client.
All user data rendered as text (§11.5).

## Scope

- `web/src/pages/Library.tsx`, `web/src/pages/Book.tsx` + components
  (`BookCard`, `HighlightCard`, `TagEditor`) + tests where noted.

## Detailed Requirements

1. Library (`/`):
   - Fetch `GET /api/books?sort=`; card shows title, author, highlight count; click →
     `/books/:id`. Sort toggle `Title | Recent` (persist in `localStorage`).
   - "Import from Apple Books" button: confirm dialog ("Read highlights from this
     Mac's Apple Books?") → `POST /api/imports {source:'apple_books'}`; while
     running: button disabled + spinner; result toast summarizing stats
     (`+12 highlights, 3 updated, 1 archived`); error toast shows envelope message
     (e.g., Full Disk Access remediation). 409 → toast "An import is already
     running".
   - Empty library → `EmptyState` with a hint to press the import button.
2. Book detail (`/books/:id`):
   - Header: title, author, count. 404 → ErrorBox with a back link.
   - Highlights via `GET /api/highlights?bookId=` (paginate with a "Load more"
     button using limit/offset; page size 50).
   - `HighlightCard`: text (multiline preserved via CSS `white-space: pre-wrap`),
     note (visually distinct), chapter, style color chip: the API supplies
     `styleName` (DESIGN.md §6.4); the UI's local `STYLE_COLORS` maps
     `styleName → CSS color` (presentation only — the UI never re-derives names
     from raw style integers), unknown names → neutral gray chip showing the name
     text; date.
   - Favorite star: optimistic toggle calling `PATCH /api/highlights/:id`; revert +
     error toast on failure.
   - `TagEditor`: chips with remove `×`; input adds on Enter; every change sends the
     **full** resulting set via `PUT /api/highlights/:id/tags`; `INVALID_TAG` →
     inline error, state reverted.
   - "Show archived" toggle appends `archived=true` results in a separated, visually
     muted section (default hidden).
3. All lists keyed by stable IDs; no `dangerouslySetInnerHTML`; user text only in
   text nodes/attributes.

## Acceptance Criteria

- Manual walkthrough against a seeded server: browse, sort, open book, paginate,
  favorite on/off, add/remove tags (incl. CJK tag), archived toggle, import button
  happy path + error path (missing container) — all behave as specified.
- Component tests (vitest + @testing-library/react, jsdom): `HighlightCard` renders
  hostile text (`<img src=x onerror=…>`) as literal text; `TagEditor` sends the full
  tag set on add and remove.
- `npm run build` stays clean.

## Validation

- `npm test`; manual checklist recorded in the PR description.

## Dependencies

- 21, 22, 23

## Non-goals

- Search/tags/favorites pages (25), highlight text editing, bulk actions.

## Design References

- DESIGN.md §10.3, §11.5

# Title

Web UI: search, tags, and favorites views

## Summary

Implement the Search view (`/search`) with query box, snippet results and filters,
the Tags view (`/tags`), and the Favorites view (`/favorites`).

## Context

DESIGN.md §10.3 route table; read endpoints from issue 21; interaction components
(favorite star, tag chips) shared from issue 24.

## Scope

- `web/src/pages/Search.tsx`, `Tags.tsx`, `Favorites.tsx` + a shared
  `HighlightResultList` component.

## Detailed Requirements

1. Search (`/search?q=`):
   - Query box synced with the `q` URL param (shareable/bookmarkable); debounce
     300 ms. Mode rules: empty `q` with no filters → idle EmptyState (no request);
     empty `q` with tag/favorite filters → **list mode**
     (`GET /api/highlights` without `q`); non-empty `q` → search mode.
   - Calls `GET /api/highlights?q=...` (search mode); renders snippets converting
     `«»` markers to `<mark>` elements by splitting the string — never HTML
     injection; result rows link to `/books/:id`.
   - Filter controls: favorite-only checkbox, tag dropdown (options from
     `GET /api/tags`); filters reflected in URL params.
   - Pagination via "Load more"; `EMPTY_QUERY`/no-hits → EmptyState.
2. Tags (`/tags`): list from `GET /api/tags` (name + count, ordering as delivered);
   click → `/search?tag=<name>` view listing that tag's highlights (list mode, no
   `q`); the Search page must therefore also handle tag-only mode (no query → list
   filtered highlights).
3. Favorites (`/favorites`): `GET /api/highlights?favorite=true` paginated; reuses
   `HighlightResultList` with favorite/tag interactions live (un-favoriting removes
   the row after a toast, not silently).
4. `HighlightResultList` accepts a normalized row type `UIHighlightRow = { id,
   bookId, bookTitle, textOrSnippet, isSnippet: boolean, favorite, tags }` with two
   mapper functions (from `SearchHit` and from `HighlightItem`) so list mode and
   search mode render through one component; `isSnippet` controls whether `«»`
   markers are converted to `<mark>`.
5. All three pages handle loading (Spinner) and `ApiError` (ErrorBox) states.
6. Strict text-node rendering for all user data; the only markup transformation is
   the `«»`→`<mark>` split described above (unit-tested against hostile snippets
   containing `<script>` and unbalanced markers).

## Acceptance Criteria

- Manual walkthrough: CJK and ASCII searches show marked snippets; tag filter and
  favorite filter compose; tags page navigates into filtered lists; favorites
  reflect toggles immediately.
- Component test: snippet renderer with `«evil»<script>alert(1)</script>` renders
  the script tag as literal text and only the `«evil»` span as `<mark>`.
- URL params round-trip (reload restores query + filters).
- `npm run build` clean; no new dependencies.

## Validation

- `npm test`; manual checklist recorded in the PR description.

## Dependencies

- 21, 22, 23 (24 for shared components)

## Non-goals

- Saved searches, keyboard shortcuts, export from UI (v2).

## Design References

- DESIGN.md §7 (snippet contract), §10.3, §11.5

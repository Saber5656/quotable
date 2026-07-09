# Title

Tag repository and tag-assignment operations

## Summary

Implement `src/core/repo/tags.ts`: tag creation with normalization, listing with
usage counts, and assignment/removal/replacement on highlights.

## Context

Tags and favorites are the only quotable-owned metadata in v1 (DESIGN.md §5).
`setFavorite` already lives in the highlights repo (issue 06); this issue owns
everything tag-shaped. Consumers: CLI 17, API 22, export 18.

## Scope

- `src/core/repo/tags.ts` + unit tests.

## Detailed Requirements

1. Normalization helper `normalizeTagName(raw: string): { name: string; norm: string }`
   implementing exactly DESIGN.md §5 "Tag normalization":
   - `name` = `raw.normalize('NFKC').trim()` with internal whitespace collapsed to
     single spaces; `norm` = `name.toLowerCase()`.
   - Reject (`AppError('INVALID_TAG')`): empty after trim, > 64 code points, names
     containing `,` or `#` or control characters.
2. `ensureTag(db, raw): TagRow` — find by `name_norm` or create (first spelling wins
   as display name).
3. `addTags(db, highlightId, raws: string[]): string[]` — idempotent; returns final
   tag display names of the highlight.
4. `removeTags(db, highlightId, raws: string[]): string[]` — removing an unassigned
   tag is a no-op; returns final tag set.
5. `replaceTags(db, highlightId, raws: string[]): string[]` — set semantics inside a
   transaction (used by `PUT /api/highlights/:id/tags`).
6. `listTags(db): { id, name, count }[]` — count = non-archived highlights carrying
   the tag; ordered by count desc then name asc. Tags with zero usage remain listed.
7. Assignment functions take **exact highlight IDs only** and throw
   `AppError('NOT_FOUND')` when the highlight does not exist. Prefix resolution is
   the CLI caller's job (`resolveHighlightId` from issue 06); no prefix logic here.
8. Unused-tag garbage collection: **none** in v1 (explicit; tags persist).

## Acceptance Criteria

- `ensureTag(' 読書 ')` and `ensureTag('読書')` return the same row; NFKC case
  (full-width `Ｂｏｏｋ` vs `Book`) unifies.
- add/remove/replace idempotency and transactionality tested (replace with a failing
  tag name mid-list leaves prior state).
- Counts exclude archived highlights.
- Invalid names rejected with `INVALID_TAG`.

## Validation

- `npm test`.

## Dependencies

- 03, 06

## Non-goals

- Favorites (in 06), tag renaming/merging (v2), tag deletion command (v2).

## Design References

- DESIGN.md §4, §5

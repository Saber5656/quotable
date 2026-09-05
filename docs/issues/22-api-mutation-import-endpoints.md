# Title

API mutation and import endpoints: favorite, tags, imports

## Summary

Implement `PATCH /api/highlights/:id`, `PUT /api/highlights/:id/tags`,
`POST /api/imports`, `GET /api/imports` on the hardened server.

## Context

DESIGN.md §10.2. Mutations only touch quotable-owned fields (§5). Import runs the
same pipeline as the CLI (issues 10–13) synchronously and is guarded against
concurrent runs via issue 08.

## Scope

- `src/server/routes/highlights.ts` (mutations), `imports.ts` + inject tests.

## Detailed Requirements

1. `PATCH /api/highlights/:id` body `{ favorite: boolean }` (exactly this one field;
   others → 400): 404 on unknown id; returns the updated `HighlightItem`.
2. `PUT /api/highlights/:id/tags` body `{ tags: string[] }` (each 1–64 chars):
   `replaceTags` semantics (issue 07); invalid tag name → 400 `INVALID_TAG` and no
   partial application; returns `{ highlightId, tags: string[] }`.
3. `POST /api/imports` body `{ source: 'apple_books', dryRun?: boolean }`:
   - If `getActiveRun` reports a running import → 409 `IMPORT_RUNNING`.
   - Runs locate→open→validate→extract→pipeline in-process (same composition as
     issue 14, factored into a shared `src/import/appleBooksImport.ts` used by both
     CLI and API — refactor issue 14's command to call it if needed).
   - Test seam: `buildServer` deps gain an optional
     `runAppleBooksImport?: (opts: { dryRun: boolean }) => ImportReport` — defaults
     to the real composition; tests inject fakes (deferred/throwing). The public
     HTTP API body never exposes paths like `dbDir`.
   - Success → 200 `ImportReport` JSON. Failures map: `APPLE_BOOKS_NOT_FOUND` → 404,
     `FULL_DISK_ACCESS_REQUIRED` → 403, `APPLE_BOOKS_SCHEMA_MISMATCH` → 422 (all with
     the standard error envelope; remediation text in `message`).
4. `GET /api/imports?limit=1..100` → `{ items: ImportRunRow[] }` newest first
   (stats parsed, error message included, stacks never).
5. Concurrency: the server runs one import at a time; a simple in-process mutex plus
   the DB-level active-run check (belt and suspenders; both tested).

## Acceptance Criteria

- Favorite toggle and tag replace round-trip via inject; unknown fields/ids rejected
  per envelope.
- Tag replace with one invalid name → 400 and prior tag set intact.
- `POST /api/imports` with an injected fake `runAppleBooksImport` returns the fake's
  stats as 200; with an injected **deferred** fake (resolves on demand), a second
  request issued while the first is in flight → one 200 and one 409 (deterministic,
  no timing races); a seeded `running` row in `import_runs` also yields 409.
- Failure mapping proven with injected throwing fakes for all three documented
  codes: `APPLE_BOOKS_NOT_FOUND` → 404, `FULL_DISK_ACCESS_REQUIRED` → 403,
  `APPLE_BOOKS_SCHEMA_MISMATCH` → 422.
- All mutations blocked without JSON content type (inherited check; one assertion
  each).

## Validation

- `npm test`.

## Dependencies

- 07, 08, 13, 20 (14 for the shared composition helper)

## Non-goals

- Async/background imports, progress streaming (v2), deleting highlights (no such
  API in v1).

## Design References

- DESIGN.md §5, §6.6–6.7, §10.2, §12

# Title

Apple Books runtime schema validation

## Summary

Implement `src/sources/appleBooks/validate.ts`: a runtime check of the opened Apple
Books databases against the declared `APPLE_BOOKS_SCHEMA` spec (owned by issue 09)
that fails with a rich diagnostic on mismatch.

## Context

Apple's schema is undocumented and may drift with OS updates (ADR-003 accepted risk).
This layer converts silent breakage into `APPLE_BOOKS_SCHEMA_MISMATCH` (exit 5) with
enough detail for a user bug report (DESIGN.md §6.3, §12; known unknown U1).

## Scope

- `validate.ts` + tests. The spec constant `APPLE_BOOKS_SCHEMA` is **owned by issue
  09** (`src/sources/appleBooks/schemaSpec.ts`); this issue consumes it.

## Detailed Requirements

1. Import `APPLE_BOOKS_SCHEMA` from `./schemaSpec.js` (created in issue 09); do not
   redefine it.
2. `validateAppleBooksSchema(db, { hasLibrary: boolean }): void`:
   - Annotation DB: table existence via `main.sqlite_master`, columns via
     `PRAGMA main.table_info('ZAEANNOTATION')`. Library DB (when
     `hasLibrary: true`): table via `lib.sqlite_master`, columns via
     `PRAGMA lib.table_info('ZBKLIBRARYASSET')` (attached DBs have their own
     `sqlite_master`). Extra unknown columns are fine.
   - On any missing table/column: throw `AppError('APPLE_BOOKS_SCHEMA_MISMATCH')`
     with `details = { missing: [...], found: { <table>: [columns...] } }` and a
     message instructing the user to run `quotable import apple-books --json` and
     attach the output to a GitHub issue.
3. A unit test must enforce that every column referenced in `extract.ts` SQL (issue
   12) appears in `APPLE_BOOKS_SCHEMA` — implement as an exported
   `REQUIRED_BY_EXTRACTION` list in `extract.ts` compared against the spec (the test
   fails if the two drift). For this issue, create the test scaffold with a
   placeholder that issue 12 fills.
4. Validation runs on the read-only extraction connection; it performs only PRAGMAs
   and `sqlite_master` reads.

## Acceptance Criteria

- Valid fixture passes; fixture with a renamed `ZANNOTATIONUUID` column (builder
  option or manual ALTER in test) fails with `missing` containing exactly
  `ZAEANNOTATION.ZANNOTATIONUUID` and `found` populated.
- Missing whole table and missing-library-with-`hasLibrary:false` (skips library
  checks) covered.
- Diagnostic message contains no absolute user paths other than the DB location
  already known to the user (privacy of the report text).

## Validation

- `npm test` against issue 09 fixtures (including a corrupted-schema variant).

## Dependencies

- 09, 10

## Non-goals

- Data extraction (12), auto-adaptation to schema changes (explicitly rejected —
  fail loudly instead).

## Design References

- DESIGN.md §6.3, §12; ADR-003; research doc "Verification status"

# Title

Synthetic Apple Books fixture generator

## Summary

Build a test utility that generates synthetic `AEAnnotation*.sqlite` and
`BKLibrary*.sqlite` files matching the researched Apple Books schema, plus a standard
fixture dataset with all edge cases. All importer issues (10–14) and the smoke test
(27) test against these fixtures.

## Context

The real Apple Books schema is undocumented and no live database was available during
research (`docs/research/apple-books-schema.md`, "Verification status"). Fixtures
encode our schema assumptions as executable artifacts; when reality diverges, fixing
the fixtures + validation layer is the upgrade path.

## Scope

- `test/fixtures/appleBooks/builder.ts` (programmatic builder)
- `test/fixtures/appleBooks/standard.ts` (the standard dataset described below)
- `src/sources/appleBooks/schemaSpec.ts` — the `APPLE_BOOKS_SCHEMA` constant
  (tables + required columns per the research doc). This issue **owns** the file;
  issue 11 consumes it for runtime validation. Shape:
  ```ts
  export const APPLE_BOOKS_SCHEMA = {
    annotation: { table: 'ZAEANNOTATION', columns: [
      'ZANNOTATIONASSETID','ZANNOTATIONUUID','ZANNOTATIONSELECTEDTEXT',
      'ZANNOTATIONNOTE','ZFUTUREPROOFING5','ZANNOTATIONSTYLE','ZANNOTATIONTYPE',
      'ZANNOTATIONDELETED','ZANNOTATIONLOCATION','ZPLLOCATIONRANGESTART',
      'ZANNOTATIONCREATIONDATE','ZANNOTATIONMODIFICATIONDATE'] },
    library: { table: 'ZBKLIBRARYASSET', columns: [
      'ZASSETID','ZTITLE','ZAUTHOR','ZGENRE','ZLANGUAGE'] },
  } as const;
  ```
- Tests proving the builder emits what the research doc / spec specifies.

## Detailed Requirements

1. Builder API:
   ```ts
   interface FixtureBook { assetId: string; title?: string | null; author?: string | null;
     genre?: string | null; language?: string | null; inLibrary?: boolean /* default true */ }
   interface FixtureAnnotation { uuid?: string | null; assetId: string;
     selectedText?: string | null; note?: string | null; chapter?: string | null;
     style?: number | null; type?: number /* default 2 */; deleted?: 0 | 1;
     location?: string | null; locationStart?: number | null;
     createdAppleTime?: number | null; modifiedAppleTime?: number | null; }
   buildAppleBooksFixture(dir: string, books: FixtureBook[],
     annotations: FixtureAnnotation[],
     opts?: { withWal?: boolean }): { containerDir: string }
     // creates <dir>/AEAnnotation/AEAnnotation_v10312011_1727_local.sqlite
     // and <dir>/BKLibrary/BKLibrary-1-091020131601.sqlite
   ```
2. Emitted schema must include every table/column named in the research doc's
   `ZAEANNOTATION` and `ZBKLIBRARYASSET` tables (extra Core Data columns like `Z_PK`,
   `Z_ENT`, `Z_OPT` included with dummy values so validation code cannot cheat).
   Books with `inLibrary: false` get annotations but no `ZBKLIBRARYASSET` row (orphan
   case).
3. Standard dataset (`standard.ts`) — exported as data + a `writeStandardFixture(dir)`
   helper; must contain at least:
   - 3 library books: JP novel (CJK title/author, 5 highlights incl. notes), EN
     non-fiction (3 highlights, one with emoji), book with zero highlights;
   - 1 orphan assetId with 2 highlights (no library row);
   - 1 annotation with `deleted: 1` for an existing book;
   - 1 annotation with `uuid: null` (fallback dedup key path);
   - 1 bookmark row (`selectedText: null`) and 1 blank-text row (must be filtered);
   - 1 highlight with exactly 64 KiB text (boundary, importable) and 1 with text
     over the cap (65 537+ code units — must be skipped and counted in
     `skippedInvalid` by issue 12);
   - 1 highlight with `locationStart: null`;
   - styles covering 0–5 and an unknown style 9;
   - Apple-epoch timestamps including `null` and fractional seconds.
4. The builder writes plain SQLite by default. With `{ withWal: true }` it must
   produce genuinely uncheckpointed WAL content so issue 10 can prove checkpoint
   recovery: create the DB, `PRAGMA journal_mode=WAL` + `PRAGMA
   wal_autocheckpoint=0`, insert a designated marker annotation row (exported
   constant `WAL_ONLY_MARKER_TEXT`), then **while the connection is still open**
   copy `{db, -wal, -shm}` into the fixture location, then close the original. The
   fixture's main DB file must not contain the marker row (verified by opening just
   the copied main file without sidecars in a self-test).
5. Fixture files are generated into temp dirs at test time — never committed binaries.

## Acceptance Criteria

- Generated DBs open with better-sqlite3; `PRAGMA table_info` contains every column
  the research doc lists, exact names.
- Standard dataset row counts and edge cases assertable via exported constants
  (e.g., `STANDARD_EXPECTED.importableHighlights`).
- Orphan, deleted, null-uuid, bookmark, blank-text cases each present exactly as
  specified.

## Validation

- `npm test` — builder self-tests + a golden test asserting every table/column in
  `APPLE_BOOKS_SCHEMA` exists in the generated DBs (via `PRAGMA table_info`).

## Dependencies

- 01 (03 useful but not required)

## Non-goals

- Real Apple data, PDF annotation fixtures, importer logic.

## Design References

- `docs/research/apple-books-schema.md`; DESIGN.md §13; ISSUE_PLAN.md §6, §8 U1

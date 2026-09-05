# Title

Apple Books extraction query and row mapping

## Summary

Implement `src/sources/appleBooks/extract.ts` and finalize `types.ts`: run the
extraction SQL over the opened read-only connection and map raw rows to validated
`SourceHighlight` DTOs with all transforms (Apple epoch, trimming, orphan books,
length caps, dedup key input fields).

## Context

This is the parser boundary for untrusted file content (DESIGN.md §6.4, §11.4 B2).
Field mapping and filtering rules are fully specified in §6.4 and the research doc.

## Scope

- `extract.ts`, `types.ts` + tests against issue 09's standard fixture.

## Detailed Requirements

1. `types.ts`:
   ```ts
   interface SourceBookInfo { assetId: string; title: string | null;
     author: string | null; genre: string | null; language: string | null; }
   interface SourceHighlight { uuid: string | null; assetId: string; text: string;
     note: string | null; chapter: string | null; location: string | null;
     locationStart: number | null; style: number | null; deletedAtSource: boolean;
     createdAt: string | null; modifiedAt: string | null; book: SourceBookInfo; }
   interface ExtractionResult { highlights: SourceHighlight[]; skippedInvalid: number;
     warnings: string[]; }
   ```
2. SQL: the reference query of the research doc, which includes
   `b.ZGENRE AS genre` and `b.ZLANGUAGE AS language` (LEFT JOIN
   `lib.ZBKLIBRARYASSET` when attached; without library DB, select annotation
   columns only and set book fields null). SQL-level filter:
   `ZANNOTATIONSELECTEDTEXT IS NOT NULL AND TRIM(ZANNOTATIONSELECTEDTEXT) <> ''`.
   Deleted rows are **included** (pipeline archives them). Ordering
   `ZANNOTATIONASSETID, ZPLLOCATIONRANGESTART`.
3. Export `REQUIRED_BY_EXTRACTION: { table: string; column: string }[]` listing every
   source column the SQL touches (including `ZGENRE` and `ZLANGUAGE`); issue 11's
   drift test compares it with `APPLE_BOOKS_SCHEMA`.
4. Row mapping per DESIGN.md §6.4 table, plus type/length hardening (§11.4):
   - Strings type-checked (`typeof === 'string'`), numbers finite; wrong-typed values
     → row skipped, `skippedInvalid++`, one aggregated warning.
   - Length caps: `text`/`note` 65_536 code units, others 4_096; over-cap → skip+count
     (never truncate silently).
   - `assetId` NULL/empty → skip+count.
   - Timestamps via `appleEpochToIso`; non-finite → null.
   - Trim `text`, `note`, `chapter`; empty-after-trim `note`/`chapter` → null.
   - `deletedAtSource = (ZANNOTATIONDELETED === 1)`.
5. Orphan handling: `book.title = null` marks an orphan; the *pipeline* renders the
   synthetic `Unknown book (<assetId>)` title — extraction passes nulls through.
6. Style constant `APPLE_STYLE_NAMES: Record<number,string>` (0 underline, 1 green,
   2 blue, 3 yellow, 4 pink, 5 purple) and formatter
   `styleName(style: number | null): string | null` (unknown → `unknown(<n>)`) are
   created by this issue in **`src/core/highlightStyle.ts`** (core, so export/CLI
   consume it without depending on `sources/` — DESIGN.md §6.4); documented as
   unverified (U2).
7. Pure module: no quotable-DB access, no filesystem writes, no console output
   (warnings returned in `ExtractionResult`).

## Acceptance Criteria

- Standard fixture yields exactly `STANDARD_EXPECTED.importableHighlights` DTOs;
  bookmark and blank-text rows absent; deleted row present with
  `deletedAtSource: true`; null-uuid row present with `uuid: null`.
- CJK/emoji text round-trips byte-identically (after trim).
- 64 KiB fixture text passes (== cap); a >cap synthetic row is skipped and counted.
- Apple epoch `0` → `2001-01-01T00:00:00.000Z`; `null` and `NaN` → null.
- Orphan annotations carry `book.title === null`.
- Drift test between `REQUIRED_BY_EXTRACTION` and `APPLE_BOOKS_SCHEMA` passes and is
  proven to fail when a column is removed from the spec (meta-test or reviewed
  manually).

## Validation

- `npm test` against fixtures (standard + hostile variants built inline).

## Dependencies

- 09, 10, 11

## Non-goals

- Upserting into quotable's DB (13), style display in UI (24).

## Design References

- DESIGN.md §6.4, §6.5 (inputs), §11.4; research doc reference query

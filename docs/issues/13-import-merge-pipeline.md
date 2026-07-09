# Title

Import merge/dedup pipeline

## Summary

Implement `src/import/pipeline.ts`: consume `SourceHighlight[]`, upsert books, apply
the dedup/update/archive decision table inside one transaction, support dry-run, and
produce an `ImportStats` report tied to an import-run row.

## Context

This is the heart of import idempotency (DESIGN.md §6.5–6.7, §5). It is
source-agnostic: nothing Apple-specific may leak in (future sources reuse it).

## Scope

- `pipeline.ts` + scenario tests. CLI wiring is issue 14; API wiring issue 22.

## Detailed Requirements

1. Export:
   ```ts
   interface ImportInput { source: 'apple_books'; highlights: SourceHighlight[];
     skippedInvalid: number; warnings: string[]; }
   interface ImportOptions { dryRun?: boolean; now?: () => string; }
   interface ImportReport { runId: string; dryRun: boolean; stats: ImportStats;
     warnings: string[]; }
   runImport(db, input: ImportInput, opts?: ImportOptions): ImportReport
   ```
2. Run-row lifecycle (one exact flow for both real and dry runs): `startRun`
   (issue 08) commits the `running` row **outside** the data transaction → open one
   better-sqlite3 transaction for all data changes → per-row logic → real run:
   commit, then `finishRun(runId, stats)`; dry run: roll back the data transaction,
   then `finishRun(runId, stats)` with the computed stats (`dry_run=1` was set at
   `startRun`). The run row is never inside the rolled-back transaction. On error:
   roll back, then `failRun(runId, message)`.
3. Dedup key (DESIGN.md §6.5): `uuid` when non-empty, else
   `'h:' + sha256hex(assetId + '\n' + (location ?? '') + '\n' + text)` (Node
   `crypto`). Implement as exported `dedupKey(h: SourceHighlight): string`.
4. Book resolution: orphan (title null) → title `Unknown book (<assetId>)`,
   author null. Upsert via issue 06; count `booksAdded`/`booksUpdated`.
5. Decision table per row (stats bucket in brackets) — exactly DESIGN.md §6.6:
   - not existing, not deletedAtSource → insert [added]; `imported_at = now`,
     `highlighted_at = createdAt`, `source_modified_at = modifiedAt`, `archived=0`.
   - not existing, deletedAtSource → nothing [skippedDeleted].
   - existing, deletedAtSource → `setArchived(true)` if not already [archived];
     if already archived → [unchanged].
   - existing, not deletedAtSource →
     if `modifiedAt` strictly newer than stored `source_modified_at` (or stored is
     NULL and `modifiedAt` non-null) → update source-owned fields + `archived=0`
     [updated]; if both timestamps NULL and content differs
     (text/note/chapter/style/location) → update [updated]; else no-op [unchanged].
   - `favorite` and tag links are never read or written by the pipeline (test-proven).
6. `skippedInvalid` from extraction is passed through into stats; pipeline adds its
   own only if book upsert fails per-row (then also `warnings` entry; row skipped).
7. Local highlights absent from the input snapshot: untouched (asserted by test).
8. Any thrown error → transaction rollback + `failRun` with the message; the
   `AppError` is rethrown for the caller.

## Acceptance Criteria

Scenario tests (standard fixture → extraction → pipeline):

- Fresh import: stats match `STANDARD_EXPECTED`; re-run immediately → everything
  [unchanged], zero [added].
- Source text edit (bump modifiedAt in fixture) → [updated], local favorite/tags on
  that highlight preserved.
- Source delete → [archived]; re-run → [unchanged]; archived row keeps its tags.
- Older `modifiedAt` than stored → [unchanged] (no downgrade).
- Null-uuid highlight dedups by hash across re-runs (no duplicates).
- Dry-run: stats identical to a real run, DB row counts unchanged afterwards, run row
  recorded with `dry_run=1`.
- Failure injection (e.g., break FK mid-run in test) → rollback + failed run row.

## Validation

- `npm test` scenario suite; each scenario asserts full stats object equality.

## Dependencies

- 06, 08, 12

## Non-goals

- Apple-specific IO (10–12), CLI/API surfaces (14, 22), scheduling.

## Design References

- DESIGN.md §5, §6.5–6.7

# Title

Apple Books database locator and safe read-only opener

## Summary

Implement `src/sources/appleBooks/locate.ts` and `open.ts`: find the AEAnnotation and
BKLibrary SQLite files (default container or `--db-dir` override), copy them with WAL
sidecars to a private temp dir, open the copies read-only, ATTACH the library DB, and
guarantee cleanup.

## Context

Never touching Books.app's live files is a core safety property (DESIGN.md §6.1–6.2,
§11.4; ADR-003). Error taxonomy feeds the CLI exit codes (§12, issue 05).

## Scope

- `locate.ts`, `open.ts`, `types.ts` (DTO stubs) + tests against issue 09 fixtures.

## Detailed Requirements

1. `locate.ts` exports
   `locateAppleBooks(opts: { dbDir?: string }): { annotationDb: string; libraryDb: string | null }`:
   - Root = `opts.dbDir` (must exist and be a directory, else
     `AppError('INVALID_DB_DIR')`) or
     `~/Library/Containers/com.apple.iBooksX/Data/Documents`.
   - Annotation DB: files under `<root>/AEAnnotation/` whose basename starts with
     `AEAnnotation` and ends with `.sqlite` (implemented with `fs.readdirSync` +
     string checks — no glob dependency, DESIGN.md §11.6); multiple → newest mtime.
     None → `AppError('APPLE_BOOKS_NOT_FOUND')` whose message names the searched
     path.
   - Library DB: same matching for `<root>/BKLibrary/BKLibrary*.sqlite`; none →
     return `null` (caller warns and proceeds with orphan-book rule).
   - Root exists but reading throws `EPERM`/`EACCES` →
     `AppError('FULL_DISK_ACCESS_REQUIRED')` with remediation text: System Settings →
     Privacy & Security → Full Disk Access → enable for the terminal app.
   - Root missing entirely → `APPLE_BOOKS_NOT_FOUND` (message distinguishes
     "container missing — has Books.app ever been used?").
2. `open.ts` exports
   `openAppleBooksReadOnly(paths): { db: Database; cleanup(): void }`:
   - Create temp dir `fs.mkdtempSync(os.tmpdir() + '/quotable-abx-')`, mode `0700`.
   - Copy annotation db + existing `-wal`/`-shm` sidecars; same for library db when
     present. Copy failures map to `FULL_DISK_ACCESS_REQUIRED` (EPERM/EACCES) or
     rethrow.
   - WAL recovery requires write access, so it runs on the **temp copies only** (they
     are quotable-private; the source files are only ever read): open each copy
     writable once, run `PRAGMA wal_checkpoint(TRUNCATE)`, close it, then reopen the
     annotation copy with `new Database(path, { readonly: true })` for extraction.
     Extraction connections are always read-only.
   - `PRAGMA trusted_schema=OFF` is executed on **every** connection to Apple DB
     copies — the one-time checkpoint connections and the extraction connection
     (DESIGN.md §11.4). Never `PRAGMA writable_schema`; no user-defined functions.
   - When library DB present, attach it read-only via a URI:
     `ATTACH DATABASE ? AS lib` with the bound parameter
     `file:<absolute-temp-copy-path>?mode=ro` (URI filename), so the attached DB
     cannot be written even though ATTACH ignores the main handle's readonly flag.
   - If any step (copy, checkpoint, open, attach) fails, the function must remove
     the temp dir before rethrowing (internal try/catch — no leaked temp dirs on
     partial failure).
   - `cleanup()` closes the connection and removes the temp dir recursively;
     idempotent; callers use try/finally.
3. No module reads quotable's own DB; no network; no writes outside the temp dir.

## Acceptance Criteria

- Fixture container → both files located; newest-mtime rule proven with two matching
  files; `--db-dir` override works.
- WAL fixture (`withWal: true`): rows added only to the WAL are visible after open
  (checkpoint worked), and the **source** files' bytes and mtimes are unchanged.
- Missing annotation dir → `APPLE_BOOKS_NOT_FOUND` (exit 3 via CLI mapping); chmod-0
  root (test-only) → `FULL_DISK_ACCESS_REQUIRED`.
- Missing library DB → `{ libraryDb: null }` path works end-to-end.
- `trusted_schema` is OFF on the extraction connection (assert via
  `PRAGMA trusted_schema`); an `UPDATE lib.ZBKLIBRARYASSET ...` attempt on the
  extraction connection throws (read-only attach proven).
- Injected failure after copy (e.g., unreadable second file) → no temp dir left
  behind.
- After `cleanup()`, temp dir is gone; double-cleanup safe.

## Validation

- `npm test` on macOS and Linux CI (fixtures make this OS-independent; the default
  container path branch is unit-tested with an injected home dir).

## Dependencies

- 02, 05 (AppError codes), 09

## Non-goals

- Schema validation (11), extraction (12), live-DB manual verification (14).

## Design References

- DESIGN.md §6.1, §6.2, §11.4, §12; ADR-003

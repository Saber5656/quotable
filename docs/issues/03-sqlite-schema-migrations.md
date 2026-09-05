# Title

SQLite connection, migration runner, and schema v1

## Summary

Implement `src/core/db/`: open/create the quotable database with required PRAGMAs, a
minimal forward-only migration runner, and migration 001 containing the full v1 schema
(including FTS5 table and triggers).

## Context

The schema is fully specified in DESIGN.md §4 and is the contract every repository and
the import pipeline build on. FTS details also serve issue 04.

## Scope

- `src/core/db/connection.ts`, `src/core/db/migrations.ts`,
  `src/core/db/migrations/001-initial.ts`, `src/core/time.ts`, `src/core/ids.ts`
- Unit tests.

## Detailed Requirements

1. `connection.ts` exports `openDatabase(file: string): Database` (better-sqlite3):
   - Creates the file if missing. On every open (not just creation), on POSIX,
     verify the file mode is `0600` and `chmod 0o600` if not (DESIGN.md §3.1).
   - PRAGMAs on every open: `journal_mode=WAL`, `foreign_keys=ON`,
     `busy_timeout=5000`, `synchronous=NORMAL`.
   - Also export `openMemoryDatabase()` for tests.
2. `migrations.ts`:
   - `migrate(db): void` — creates `schema_migrations` if absent, applies pending
     migrations in ascending `version` inside one transaction each, records
     `applied_at` (ISO UTC).
   - Migrations are an ordered in-code array `[{ version: 1, up(db) }]`; no down
     migrations; unknown recorded versions greater than the max known → throw
     `AppError('SCHEMA_TOO_NEW')`.
3. Migration 001 executes the DDL from DESIGN.md §4 **except**
   `CREATE TABLE schema_migrations` (the runner owns that table exclusively):
   `books`, `highlights`, `tags`, `highlight_tags`, `import_runs`, indexes,
   `highlights_fts` (FTS5, `content='highlights'`, `tokenize='trigram'`), and the
   three sync triggers, verbatim including CHECK and UNIQUE constraints.
4. `time.ts`: `nowIso(): string`; `appleEpochToIso(v: number | null): string | null`
   (`v + 978307200` seconds → ISO, fractional seconds preserved to ms).
5. `ids.ts`: `newId(): string` returning a ULID (lowercase not required; use `ulid()`
   as-is).
6. If FTS5/trigram is unavailable in the bundled SQLite, fail loudly in tests (this is
   known unknown U4 — do not silently degrade).

## Acceptance Criteria

- Fresh DB → `migrate` produces all tables/indexes/triggers;
  `PRAGMA foreign_keys` = 1; `journal_mode` = `wal`; file mode `0600`.
- Running `migrate` twice is a no-op (idempotent).
- Insert into `highlights` → row findable via
  `SELECT rowid FROM highlights_fts WHERE highlights_fts MATCH '"..."'` for a CJK
  substring of length 3 (proves trigram tokenizer works).
- Update/delete of a highlight keeps FTS in sync (trigger tests).
- `appleEpochToIso(0)` = `2001-01-01T00:00:00.000Z`; `null` → `null`.

## Validation

- `npm test` with the cases above, using temp-file and in-memory DBs.

## Dependencies

- 01, 02

## Non-goals

- Repository/query logic (06, 07), search API (04), any data seeding.

## Design References

- DESIGN.md §3.2, §4; ISSUE_PLAN.md §8 U4

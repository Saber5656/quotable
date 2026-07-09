# ADR-003: Import by reading Apple Books' SQLite directly (macOS-only v1)

Status: Accepted (2026-07-08, confirmed with product owner)

## Context

Apple Books provides no export API. Its highlights live in undocumented Core Data
SQLite databases inside the Books.app sandbox container on macOS
(`AEAnnotation*.sqlite`, `BKLibrary*.sqlite` — see `docs/research/apple-books-schema.md`).
Options: read those databases directly, require the user to manually share/export from
Books, or scrape UI. The product owner accepted macOS-only v1 with direct reads.

## Decision

The importer locates the databases by glob under
`~/Library/Containers/com.apple.iBooksX/Data/Documents/`, copies `{db, -wal, -shm}` to
a private temp directory, opens the copies read-only, and extracts via SQL. A runtime
schema-validation layer checks every referenced table/column before extraction and
fails with a diagnostic (`APPLE_BOOKS_SCHEMA_MISMATCH`) on drift. `--db-dir` allows
pointing at a backup copy of the container structure.

## Consequences

- Best UX (one command, no manual steps), fully local.
- **Accepted risk**: Apple may change the schema in any macOS update. Mitigations:
  validation layer turns silent breakage into an actionable error; the extraction
  column list lives in one module; fixtures (issue 09) make schema updates testable.
- macOS-only importer in v1; `sources/` is an interface so future sources (Kindle,
  Readwise CSV) slot in without touching the pipeline.
- Reading the container may require Full Disk Access; the importer detects `EPERM` and
  prints remediation (`FULL_DISK_ACCESS_REQUIRED`, exit code 4).
- Apple Books PDF annotations are stored differently and are explicitly out of scope
  for v1.

## Alternatives rejected

- **Manual export flow**: safe but per-book manual clicking; not automatable; poor fit
  for the product's core promise.
- **UI scripting/scraping**: brittle, permission-heavy, slow.

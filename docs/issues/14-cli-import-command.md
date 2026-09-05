# Title

CLI command: `import apple-books`

## Summary

Wire locator → opener → validation → extraction → pipeline into
`quotable import apple-books [--db-dir <path>] [--dry-run]`, with human and `--json`
run reports and the documented exit codes. Includes the live-database manual
verification checklist.

## Context

First user-facing feature (DESIGN.md §8.2, §6). All heavy lifting exists in issues
10–13; this issue is composition + UX + the real-world check for known unknowns U1–U3.

## Scope

- `src/cli/commands/import.ts` + child-process tests + manual checklist execution.

## Detailed Requirements

1. Command: `quotable import apple-books`, options `--db-dir <path>`, `--dry-run`.
   Global flags (`--data-dir`, `--json`, `--quiet`) apply.
2. Sequence: locate (warn to stderr when library DB missing) → open (try/finally
   cleanup) → validate schema → extract → `runImport` → print report.
3. Human report (stdout), single block:
   ```
   Imported from Apple Books (dry run)          ← suffix only when --dry-run
     Books:      +2 added, 1 updated
     Highlights: +12 added, 3 updated, 1 archived, 40 unchanged
     Skipped:    2 deleted-at-source, 1 invalid
     Warnings:   1 (see below)
   ```
   Warnings listed after, one per line, to stderr.
4. `--json` (stdout): `{ runId, dryRun, stats: ImportStats, warnings: string[] }` —
   exactly the `ImportReport` shape.
5. Exit codes: success 0 (even with warnings); `APPLE_BOOKS_NOT_FOUND` 3;
   `FULL_DISK_ACCESS_REQUIRED` 4; `APPLE_BOOKS_SCHEMA_MISMATCH` 5 (diagnostic printed);
   other `AppError`/unexpected 1; bad flags 2.
6. Temp-dir cleanup guaranteed on all paths including failure (test with injected
   failure after open).
7. **Manual live-DB checklist** (executed once by a human/agent on a Mac with real
   Books.app data; results recorded in the issue/PR):
   - highlight 3 passages (one with a note, one CJK) in Books.app;
   - run `quotable import apple-books --dry-run --json`, then real run;
   - verify counts, text fidelity, chapter/style values; delete one highlight in
     Books.app, re-import, verify [archived];
   - record actual macOS version + whether Full Disk Access was needed;
   - discrepancies against `docs/research/apple-books-schema.md` → update research
     doc + `schemaSpec.ts` + fixtures (or file follow-up issues).

## Acceptance Criteria

- Against fixture container (`--db-dir`): fresh import, idempotent re-run, and
  dry-run proven via `runCli` child-process tests, asserting stdout JSON and exit
  codes.
- Error-path distinction proven: `--db-dir` pointing at a nonexistent path →
  `INVALID_DB_DIR`, exit 1; `--db-dir` pointing at an existing directory without an
  `AEAnnotation/*.sqlite` → `APPLE_BOOKS_NOT_FOUND`, exit 3.
- Human output matches the format above (snapshot test).
- Manual live-DB checklist executed and recorded. If no real data is available at
  implementation time, a **blocking follow-up issue** must be filed and linked from
  issue 29's release checklist — the v1 release gate requires the live-DB evidence
  or an explicit maintainer waiver recorded there (ISSUE_PLAN.md §6).

## Validation

- `npm test`; manual checklist evidence attached to the PR.

## Dependencies

- 05, 13 (which transitively brings 09–12)

## Non-goals

- API import endpoint (22), scheduling, non-Apple sources.

## Design References

- DESIGN.md §6, §8.1, §8.2, §12; ISSUE_PLAN.md §8 U1–U3

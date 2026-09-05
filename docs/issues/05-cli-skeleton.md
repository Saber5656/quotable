# Title

CLI skeleton: global flags, output helpers, error-to-exit-code mapping

## Summary

Implement the `quotable` CLI entry point with commander: global flags, stdout/stderr
conventions, table/JSON printers, `AppError` → exit-code mapping, and the `db path` /
`db info` commands as the first real commands.

## Context

All command issues (14–19, 26) plug into this skeleton. Conventions are DESIGN.md §8.1;
error codes §12.

## Scope

- `src/cli/index.ts`, `src/cli/output.ts`, `src/cli/errors.ts`,
  `src/cli/commands/db.ts`, shared `src/core/errors.ts` (if not created in 02).
- Tests running the built CLI as a child process.

## Detailed Requirements

1. `index.ts`: commander program `quotable`, version from package.json, global options
   `--data-dir <path>`, `--json`, `--quiet`. Subcommands register from
   `src/cli/commands/*`. Unknown command/option → commander usage error → exit 2.
2. Context creation helper `createContext(globalOpts)`: resolves data dir (issue 02),
   ensures it, opens DB, runs migrations (issue 03), returns `{ db, dataDir, out }`.
   Commands that don't need the DB (e.g., `db path`) resolve paths only.
3. `output.ts`:
   - `printTable(rows: Record<string,string|number|null>[], columns)` — plain aligned
     text, no ANSI color, truncate cell display at 80 chars with `…`.
   - `printJson(value)` — `JSON.stringify(value, null, 2)` to stdout.
   - `warn/info` → stderr; suppressed by `--quiet` (errors never suppressed).
4. `errors.ts`: `AppError` has the canonical constructor
   `new AppError(code: string, message: string, details?: unknown)` — every issue
   referencing `AppError` uses this shape. Map `AppError.code` → exit codes per
   DESIGN.md §8.1
   (`USAGE`→2, `APPLE_BOOKS_NOT_FOUND`→3, `FULL_DISK_ACCESS_REQUIRED`→4,
   `APPLE_BOOKS_SCHEMA_MISMATCH`→5, anything else→1). Print `error: <message>` to
   stderr; with env `QUOTABLE_DEBUG=1` also print the stack. Process must not print
   raw uncaught stacks in normal operation.
5. `db path` → prints resolved data dir (respects `--data-dir`/env). `db info` →
   opens DB, prints schema version, counts of books/highlights/tags, DB file size;
   supports `--json` (`{ dataDir, schemaVersion, books, highlights, tags, fileBytes }`).
6. Exit code contract test harness: helper `runCli(args, opts)` used by all later CLI
   tests (spawns `node dist/cli/index.js`, captures stdout/stderr/exit code, sets
   `QUOTABLE_DATA_DIR` to a temp dir). CLI test files ensure the build exists via a
   vitest `globalSetup` that runs `npm run build` once (matches issue 01's CI order:
   build before test).

## Acceptance Criteria

- `quotable --help`, `quotable db path`, `quotable db info --json` work; JSON shape
  exactly as above.
- `quotable nope` → exit 2 with usage on stderr.
- A command throwing `AppError('FULL_DISK_ACCESS_REQUIRED')` (test stub) exits 4 with
  the message on stderr and no stack trace.
- `--data-dir` beats `QUOTABLE_DATA_DIR` (proven via `db path`).
- First `db info` run against an empty temp dir creates the DB with migrations applied.

## Validation

- `npm test` — child-process tests via `runCli` covering the criteria above.

## Dependencies

- 01, 02, 03 (`createContext` and `db info` need migrations)

## Non-goals

- Any import/list/search/export/serve command (later issues).

## Design References

- DESIGN.md §8.1, §8.2 (`db`), §12

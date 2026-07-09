# Title

Import-run recording module

## Summary

Implement `src/core/repo/importRuns.ts`: create/finish/fail import-run rows, stale-run
recovery, and listing recent runs.

## Context

Every import (CLI or API, including dry runs and failures) is recorded in
`import_runs` (DESIGN.md §6.6–6.7). The API exposes runs via `GET /api/imports` and
guards concurrent imports with it.

## Scope

- `src/core/repo/importRuns.ts` + unit tests, including the `ImportStats` type.

## Detailed Requirements

1. Types:
   ```ts
   interface ImportStats { booksAdded: number; booksUpdated: number; added: number;
     updated: number; archived: number; unchanged: number; skippedDeleted: number;
     skippedInvalid: number; }
   interface ImportRunRow { id: string; source: string; startedAt: string;
     finishedAt: string | null; status: 'running'|'success'|'failed';
     dryRun: boolean; stats: ImportStats | null; error: string | null; }
   ```
2. Internal helper `recoverStaleRuns(db, now)`: any `running` row with
   `started_at < now - 1h` is updated to `failed` with `error='stale'`. Called by
   BOTH `startRun` and `getActiveRun` before they read/write.
3. `startRun(db, source, dryRun, now?: () => string): ImportRunRow` — runs stale
   recovery, then inserts `status='running'`.
4. `finishRun(db, id, stats: ImportStats)` → `status='success'`, `finished_at`,
   `stats_json`. `failRun(db, id, errorMessage)` → `status='failed'`,
   `finished_at`, `error` (message string only, never a stack). Both require the
   target row to exist with `status='running'`; otherwise throw
   `AppError('INVALID_RUN_STATE')` (enforces the §6.7 state machine
   `running → success | failed`).
5. `getActiveRun(db, source, now?: () => string): ImportRunRow | undefined` — runs
   stale recovery first (used by the API 409 guard).
6. `listRuns(db, limit = 20): ImportRunRow[]` newest first; parses `stats_json`
   defensively (corrupt JSON → `stats: null`, never throw).

## Acceptance Criteria

- Lifecycle covered by tests: start→finish, start→fail, stats round-trip.
- Stale recovery: a `running` row 2h old becomes `failed/stale` on next `startRun`;
  a 5-minute-old one does not.
- `getActiveRun` returns the fresh running row and ignores stale ones.
- Corrupt `stats_json` in DB → `listRuns` still succeeds with `stats: null`.

## Validation

- `npm test` (inject clock via optional `now` parameter to make staleness testable).

## Dependencies

- 03

## Non-goals

- The merge pipeline itself (13), API endpoints (22).

## Design References

- DESIGN.md §4 (`import_runs`), §6.6, §6.7

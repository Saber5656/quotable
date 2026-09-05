# Title

Data directory resolution module

## Summary

Implement `src/core/paths.ts`: resolve the quotable data directory from flag/env/
default, create it with safe permissions, and expose canonical file paths.

## Context

Every storage consumer (DB, importer, server) needs one deterministic data-dir
resolution (DESIGN.md §3.1). Permissions are part of the security model (§11.6).

## Scope

- `src/core/paths.ts` + unit tests. No CLI wiring (issue 05 consumes it).

## Detailed Requirements

1. Export `resolveDataDir(opts: { flag?: string; env?: NodeJS.ProcessEnv;
   platform?: NodeJS.Platform; homedir?: string }): string` (`platform` defaults to
   `process.platform`, `homedir` to `os.homedir()` — injectable for tests):
   - Order: `opts.flag` → `env.QUOTABLE_DATA_DIR` → platform default.
   - Platform default: on `darwin`, `<homedir>/Library/Application Support/quotable`;
     otherwise `$XDG_DATA_HOME/quotable` if `XDG_DATA_HOME` set, else
     `<homedir>/.local/share/quotable`.
   - Expand a leading `~` in flag/env values; resolve to an absolute path.
   - Reject (throw `AppError` code `INVALID_DATA_DIR`) empty strings and paths that
     exist but are not directories.
2. Export `ensureDataDir(dir: string): void`: `mkdir -p` with mode `0o700`; if the
   directory already exists with looser permissions, `chmod 0o700`. On POSIX the
   final mode MUST be `0700` — verify with `fs.statSync` and throw
   `AppError('DATA_DIR_PERMS')` if it cannot be enforced (DESIGN.md §3.1/§11.6). On
   non-POSIX platforms (not shipped in v1) this check is a no-op.
3. Export `dbPath(dir: string): string` → `<dir>/quotable.db`.
4. No module-level side effects; pure functions taking inputs explicitly (testability).
5. `AppError` may be introduced here as `src/core/errors.ts` (`code`, `message`,
   `details?`) if issue 05 has not landed; keep it dependency-free.

## Acceptance Criteria

- Resolution order proven by unit tests (flag beats env beats default).
- `darwin` default is `~/Library/Application Support/quotable` (test with mocked
  `os.homedir` and `process.platform` handling via injectable params).
- `ensureDataDir` creates nested dirs with mode `0700` (assert via `fs.statSync`).
- Non-directory existing path → `AppError('INVALID_DATA_DIR')`.

## Validation

- `npm test` — unit tests covering: flag, env, default (darwin/linux), `~` expansion,
  invalid path, permission bits.

## Dependencies

- 01 (toolchain)

## Non-goals

- Config files (none in v1), CLI flag parsing, DB opening.

## Design References

- DESIGN.md §3.1, §11.6, §12

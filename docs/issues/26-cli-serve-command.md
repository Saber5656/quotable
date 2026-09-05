# Title

CLI command: `serve`

## Summary

Implement `quotable serve [--port <n>] [--open]`: start the hardened server on
127.0.0.1 serving the bundled SPA and API, with clean startup/shutdown UX.

## Context

DESIGN.md §8.2, §10.1; ADR-004 (binding is not configurable). Server construction
from issue 20; SPA bundle from issue 23.

## Scope

- `src/cli/commands/serve.ts` + tests.

## Detailed Requirements

1. Options: `--port <n>` (default 8484; integer 1024–65535 else exit 2), `--open`
   (launch `open http://127.0.0.1:<port>` on darwin; ignore elsewhere with a stderr
   note). No flag can change the bind address.
2. Resolve `webRoot = <package>/dist/web` relative to the installed module
   (`import.meta.url`), not `process.cwd()`. Missing `index.html` → warn to stderr
   ("web UI not built; API only") and pass `webRoot: null`.
3. Startup: run migrations, then `startServer(app, port)` from issue 20 (which owns
   the `127.0.0.1` binding); print exactly one
   line to stdout: `quotable serving at http://127.0.0.1:<port>` (plus a stderr hint
   `Press Ctrl+C to stop`).
4. `EADDRINUSE` → `AppError('PORT_IN_USE')`, message suggests `--port`, exit 1.
5. Graceful shutdown on SIGINT/SIGTERM: close server, close DB, exit 0 within 3s
   (force-exit fallback timer).
6. `--json` global flag: not supported here — reject with exit 2 and a message
   (long-running process has no JSON document output).

## Acceptance Criteria

- Child-process test: start on a random free port with a temp data dir; poll
  `GET /api/health` → 200; `Host: evil.com` request → 400 (hardening active
  end-to-end); SIGINT → process exits 0.
- Bind address verified as `127.0.0.1` (connect via `127.0.0.1` succeeds; server
  address reported by the OS is loopback).
- Occupied port → exit 1 with `PORT_IN_USE` message.
- `quotable serve --json` → exit 2 with the documented rejection message.
- Missing `dist/web` → API works, `/` returns 404, warning printed.

## Validation

- `npm test` (real-socket tests with short timeouts).

## Dependencies

- 05, 20, 23

## Non-goals

- Daemonization, TLS, auth (v2), remote binding (never in v1).

## Design References

- DESIGN.md §8.2, §10.1, §12; ADR-004

# Title

End-to-end smoke test

## Summary

A scripted whole-product test: install the packed package into a temp prefix, then
run the full user journey against fixture data — import → list → search → tag →
favorite → export → serve → HTTP checks. Runs in CI on macos-latest.

## Context

Per-issue tests validate modules; this proves the assembled product and the packaging
(bin wiring, bundled SPA) together (DESIGN.md §13; ISSUE_PLAN.md §6).

## Scope

- `test/e2e/smoke.test.ts` (vitest, tagged/slow) or `scripts/smoke.sh` + CI job
  step. Implementer picks one mechanism; requirements below are mechanism-neutral.

## Detailed Requirements

1. Setup: `npm run build && npm pack --pack-destination <tmp>` → capture the
   printed tarball filename → `npm install -g --prefix <tmp>/prefix
   <tmp>/quotable-<version>.tgz` → use `<tmp>/prefix/bin/quotable` for every step
   (not `dist/` directly). Temp `QUOTABLE_DATA_DIR`; fixture container from issue
   09 (`writeStandardFixture`).
2. Journey (each step asserts exit code and key output):
   1. `quotable import apple-books --db-dir <fixture> --json` → stats equal
      `STANDARD_EXPECTED`.
   2. Re-run import → all unchanged.
   3. `quotable list books --json` → expected book count incl. `Unknown book (...)`.
   4. `quotable search <CJK term from fixture> --json` → expected hit.
   5. `quotable tag add <id-prefix> reading 名著` then
      `quotable list highlights --tag 名著 --json` → 1 item.
   6. `quotable favorite add <id>` → visible via `--favorites`.
   7. `quotable export markdown --out <tmp>/vault --json` → files exist; one golden
      content check.
   8. `quotable serve --port <random>` (background): `GET /api/health` 200;
      `GET /api/books` count matches; `PATCH` favorite via curl with proper
      headers → 200; `Host: evil.com` → 400; then SIGINT → exit 0.
3. Runtime budget: < 3 minutes. After the install step completes, the journey
   itself makes no network requests beyond loopback (the tarball install may hit
   the npm registry/cache like any `npm ci`).
4. CI: runs in the `macos-latest` job after build (also runnable locally with
   `npm run test:e2e`).

## Acceptance Criteria

- The journey passes on macOS CI from a clean checkout.
- Failure of any step fails CI with the step's stdout/stderr captured.
- The test uses only the packed artifact (catches `files`/`bin`/asset-path bugs).

## Validation

- CI run green; local `npm run test:e2e` documented in README (issue 29).

## Dependencies

- 14, 15, 16, 17, 19, 21, 22, 26 (09 fixtures)

## Non-goals

- Browser automation of the SPA (manual checklists in 24/25 cover it for v1),
  performance testing.

## Design References

- DESIGN.md §13; ISSUE_PLAN.md §6

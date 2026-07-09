# quotable — v1 Issue Plan

Derived from `docs/DESIGN.md`. GitHub Issues are generated from `docs/issues/*.md`;
this file defines ordering, dependencies, waves, coverage, and validation strategy.

## 1. v1 completion statement

When every issue listed in §2 is completed and validated, quotable v1 is complete as
defined by DESIGN.md §1.3: idempotent Apple Books import on macOS, full-text search
(CJK-capable), tags and favorites, Markdown export, a hardened localhost web UI, the
security requirements of DESIGN.md §11 enforced and tested, CI green, and the package
publishable to npm under MIT. No v1 product behavior exists outside this plan.

## 2. Issue list (recommended execution order)

| # | File | Title | Wave |
|---|---|---|---|
| 01 | `issues/01-project-scaffolding.md` | Project scaffolding, toolchain, and CI | 0 |
| 02 | `issues/02-app-paths.md` | Data directory resolution module | 0 |
| 03 | `issues/03-sqlite-schema-migrations.md` | SQLite connection, migration runner, schema v1 | 0 |
| 04 | `issues/04-fts-search-module.md` | FTS5 index and search query module | 0 |
| 05 | `issues/05-cli-skeleton.md` | CLI skeleton, global flags, error/exit-code conventions | 0 |
| 06 | `issues/06-book-highlight-repos.md` | Book and highlight repositories | 1 |
| 07 | `issues/07-tag-favorite-repos.md` | Tag repository and tag assignment (favorites live in 06) | 1 |
| 08 | `issues/08-import-run-recorder.md` | Import-run recording module | 1 |
| 09 | `issues/09-apple-books-fixtures.md` | Synthetic Apple Books fixture generator | 2 |
| 10 | `issues/10-apple-books-locator-opener.md` | Apple Books locator and safe read-only opener | 2 |
| 11 | `issues/11-apple-books-schema-validation.md` | Apple Books runtime schema validation | 2 |
| 12 | `issues/12-apple-books-extraction.md` | Apple Books extraction and row mapping | 2 |
| 13 | `issues/13-import-merge-pipeline.md` | Import merge/dedup pipeline | 2 |
| 14 | `issues/14-cli-import-command.md` | CLI command: `import apple-books` | 2 |
| 15 | `issues/15-cli-list-commands.md` | CLI commands: `list books`, `list highlights` | 3 |
| 16 | `issues/16-cli-search-command.md` | CLI command: `search` | 3 |
| 17 | `issues/17-cli-tag-favorite-commands.md` | CLI commands: `tag`, `favorite` | 3 |
| 18 | `issues/18-markdown-export-module.md` | Markdown export module (rendering + path safety) | 3 |
| 19 | `issues/19-cli-export-command.md` | CLI command: `export markdown` | 3 |
| 20 | `issues/20-http-server-security.md` | HTTP server skeleton and security middleware | 4 |
| 21 | `issues/21-api-read-endpoints.md` | API: books, highlights, search, tags (read) | 4 |
| 22 | `issues/22-api-mutation-import-endpoints.md` | API: favorites, tags, imports (mutations) | 4 |
| 23 | `issues/23-web-ui-scaffold.md` | Web UI scaffold and build integration | 4 |
| 24 | `issues/24-web-ui-library-book.md` | Web UI: library and book detail views | 4 |
| 25 | `issues/25-web-ui-search-tags-favorites.md` | Web UI: search, tags, favorites, import trigger | 4 |
| 26 | `issues/26-cli-serve-command.md` | CLI command: `serve` | 4 |
| 27 | `issues/27-e2e-smoke.md` | End-to-end smoke test | 5 |
| 28 | `issues/28-security-acceptance-tests.md` | Security acceptance test suite | 5 |
| 29 | `issues/29-packaging-readme-release.md` | Packaging, README, LICENSE, release readiness | 5 |

## 3. Dependency table

| Issue | Blocked by |
|---|---|
| 01 | — |
| 02 | 01 |
| 03 | 01, 02 |
| 04 | 03 |
| 05 | 01, 02, 03 |
| 06 | 03 |
| 07 | 03, 06 |
| 08 | 03 |
| 09 | 01 |
| 10 | 02, 05 (error codes), 09 |
| 11 | 09, 10 |
| 12 | 09, 10, 11 |
| 13 | 06, 08, 12 |
| 14 | 05, 13 |
| 15 | 05, 06, 07 |
| 16 | 04, 05, 06 |
| 17 | 05, 07 |
| 18 | 06, 07, 12 |
| 19 | 05, 18 |
| 20 | 01, 03 |
| 21 | 04, 06, 07, 20 |
| 22 | 07, 08, 13, 14, 20 |
| 23 | 01, 20 |
| 24 | 21, 22, 23 |
| 25 | 21, 22, 23, 24 |
| 26 | 05, 20, 23 |
| 27 | 09, 14, 15, 16, 17, 19, 21, 22, 26 |
| 28 | 13, 18, 19, 20, 21, 22 |
| 29 | all of 01–28 |

Within a wave, issues without a mutual dependency are parallelizable
(e.g., 06/08; 15/16/17; 21/22).

## 4. Implementation waves

| Wave | Goal | Issues |
|---|---|---|
| 0 Foundation | repo builds, DB schema exists, CLI shell runs | 01–05 |
| 1 Domain | repositories and run recording over schema v1 | 06–08 |
| 2 Import | Apple Books import works end-to-end via CLI | 09–14 |
| 3 CLI + export | all read/manage/export commands | 15–19 |
| 4 Server + UI | hardened API and SPA, `serve` command | 20–26 |
| 5 Release | whole-product validation and packaging | 27–29 |

## 5. Coverage: DESIGN.md section → issues

| DESIGN.md section | Issues |
|---|---|
| §2 Architecture / stack | 01 |
| §3 Storage layout & config | 02, 03 |
| §4 Data model | 03, 04 |
| §5 Domain rules | 06, 07, 13 |
| §6 Apple Books importer (6.1–6.7) | 10 (6.1–6.2), 11 (6.3), 12 (6.4–6.5), 13 (6.6–6.7), 09 (fixtures) |
| §7 Search behavior | 04, 16, 21 |
| §8 CLI design | 05, 14, 15, 16, 17, 19, 26 |
| §9 Markdown export | 18, 19 |
| §10 HTTP server & Web UI | 20, 21, 22, 23, 24, 25, 26 |
| §11 Security model | 20 (11.3), 10+12 (11.4 parser), 04+16+21 (11.4 injection), 18+19 (11.1 B5 path safety), 24+25 (11.5), 01+29 (11.6), 28 (verification) |
| §12 Error handling | 05, 10, 20 |
| §13 Testing & validation | 09, 27, 28 + per-issue Validation sections |
| §14 Packaging & release | 29 |

## 6. Validation strategy (whole product)

- **Per issue**: every issue carries its own Validation section (unit/integration
  tests + commands to run); an issue is not done until they pass in CI.
- **Fixtures as contract**: issue 09's synthetic Apple Books DBs encode the researched
  schema, including edge cases (deleted rows, NULL UUID, orphan assets, CJK text,
  emoji, oversized text). Import issues 10–14 validate exclusively against them.
- **Security gate**: issue 28 is a dedicated adversarial suite for DESIGN.md §11.3–11.4
  (Host/Origin/DNS-rebinding, CSRF via forms, FTS injection, path traversal). Release
  (29) is blocked on it.
- **Whole-product smoke** (issue 27): scripted flow — import fixtures → list → search
  (CJK + ASCII) → tag/favorite → export markdown → serve → exercise API via HTTP —
  run in CI on macos-latest with the built package (`npm pack` install).
- **Real-data verification**: fixtures cannot prove Apple's real schema; the first
  import run against a live Books.app database is a manual checklist in issue 14 and
  a **blocking release gate** — issue 29's checklist requires the issue 14 live-DB
  evidence (or an explicit maintainer waiver recorded there) before tagging v1.0.0.
  Tracked as known unknown U1 (§8).

## 7. Deferred v2 items

Additional sources (Kindle `My Clippings.txt`, Readwise CSV, generic CSV); Apple Books
PDF annotations; custom export templates; bidirectional Markdown sync; daily
review/spaced repetition; in-app notes on highlights; scheduled/watch imports
(launchd); optional token auth for `serve`; UI localization (Japanese); binary
distribution (Homebrew).

## 8. Known unknowns (may create new issues during implementation)

| # | Unknown | Trigger | Likely action |
|---|---|---|---|
| U1 | Real Apple Books schema on current macOS differs from research (dev machine had no live DB) | first live-DB run of issue 14's manual checklist | patch `validate.ts`/`extract.ts` column lists + fixtures; new issue if structural |
| U2 | `ZANNOTATIONSTYLE` value→color mapping wrong | same manual run | one-constant fix (DESIGN.md §6.4) |
| U3 | Full Disk Access behavior varies (EPERM vs empty dir) across macOS versions | issue 10 implementation/manual test | extend error taxonomy |
| U4 | FTS5 `trigram` tokenizer unavailable in shipped better-sqlite3 build | issue 04 first test run | pin better-sqlite3 version or compile flag; ADR update |
| U5 | npm package name `quotable` unavailable | issue 29 publish dry-run | scope to `@saber5656/quotable` (DESIGN.md §14) |
| U6 | Apple Books stores some EPUB annotations with NULL `ZPLLOCATIONRANGESTART` breaking ordering | fixture/live testing | fallback ordering by created date; noted in issue 12 |

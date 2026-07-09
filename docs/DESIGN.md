# quotable — Design Document (v1)

quotable is a local-first "personal Readwise": it consolidates reading highlights into a
single local database, makes them searchable and browsable, and exports them to
Markdown. v1 imports from **Apple Books on macOS** only.

This document is the canonical design for v1. Issue drafts in `docs/issues/*.md` are
derived from it; `docs/ISSUE_PLAN.md` maps every section here to issues.

Related documents:

- `docs/research/apple-books-schema.md` — Apple Books storage research
- `docs/decisions/ADR-001..004` — recorded architecture decisions

## 1. Product definition

### 1.1 Target user & jobs

A single macOS user who reads in Apple Books and wants to:

1. Pull all highlights/notes out of Apple Books into storage they own.
2. Browse and full-text search them (including Japanese/CJK text).
3. Organize them with tags and favorites.
4. Export them as Markdown files (e.g., into an Obsidian vault).

### 1.2 Form factor

A single npm-installable CLI (`quotable`) that also ships a local web UI
(`quotable serve` starts a localhost-only HTTP server serving a bundled SPA + JSON API).

### 1.3 v1 scope (completion definition)

v1 is complete when all of the following work end-to-end:

- `quotable import apple-books` imports highlights from the local Apple Books
  databases idempotently (re-runs never duplicate; source-side edits and deletions
  are reflected via the source's modification timestamps and deleted flag — rows
  absent from the source snapshot are left untouched, see §6.6; tags/favorites
  survive re-import).
- `quotable list books|highlights`, `quotable search <q>` work with human and `--json`
  output.
- `quotable tag add|rm|list` and `quotable favorite add|rm` manage local metadata.
- `quotable export markdown --out <dir>` writes one Markdown file per book.
- `quotable serve` runs a localhost-only web UI with: library view, book detail view,
  full-text search, tag/favorite management, and an import trigger.
- Security requirements of §11 hold and are covered by automated tests.
- Test suite and CI pass; package is publishable to npm under the MIT license.

### 1.4 v1 non-goals

- Sources other than Apple Books (Kindle, Kobo, Readwise CSV, browser articles).
- Apple Books **PDF** annotations (stored differently; see research doc).
- Windows/Linux support for the importer (CLI/server/UI code must not be
  macOS-specific, but v1 only ships the Apple Books source).
- Bidirectional sync with Markdown/Obsidian (export is one-way; SQLite is canonical —
  ADR-002).
- Editing highlight text or notes inside quotable (source of truth for content is the
  reading app; quotable owns only tags/favorites in v1).
- Daily review / spaced repetition.
- Multi-user, remote access, authentication, cloud anything.
- Automatic scheduled imports (manual trigger only).

### 1.5 Deferred to v2 (explicitly)

- Additional import sources (Kindle `My Clippings.txt`, Readwise CSV, generic CSV).
- Custom export templates (`--template`), per-book export filters beyond `--book`.
- Daily review surface.
- Note-taking on highlights inside quotable.
- Watch mode / scheduled import (launchd).
- Localization of the UI (v1 UI text is English).

### 1.6 Known unknowns

Tracked in `docs/ISSUE_PLAN.md §8`; the significant ones:

- Real Apple Books schema on current macOS (research not yet verified against a live
  DB; the dev machine had no Books data). Mitigated by issue 11 (schema validation
  layer) and issue 09 (fixtures built from the researched schema).
- `ZANNOTATIONSTYLE` value→color mapping needs empirical confirmation.
- Whether Full Disk Access is required on the target macOS version and how the error
  surfaces (mitigated by issue 10's error taxonomy).

## 2. Architecture overview

### 2.1 Stack (ADR-001)

| Layer | Choice |
|---|---|
| Language | TypeScript 5.x, `strict: true`, ESM only |
| Runtime | Node.js >= 20 |
| CLI framework | `commander` |
| Storage | SQLite via `better-sqlite3` (synchronous API) |
| HTTP server | `fastify` + `@fastify/static` |
| Web UI | React 18 + `react-router-dom`, built with Vite, plain CSS |
| IDs | ULID (`ulid` package) |
| Tests | `vitest`; API tests via `fastify.inject()` |
| Lint/format | ESLint (typescript-eslint) + Prettier |

One npm package (`quotable`), one repository. The Vite project lives in `web/` and
builds into `dist/web/`, which the server serves statically. No npm workspaces.

### 2.2 Process model

Everything is one short-lived CLI process, except `quotable serve` which is a
long-lived process hosting Fastify. There are no daemons, no background jobs. Import
triggered from the web UI runs in-process in the server (synchronous, returns the run
report in the HTTP response).

### 2.3 Module map (source layout)

```
src/
  cli/
    index.ts            # entry point (bin), command registration only
    commands/
      import.ts         # `import apple-books`
      list.ts           # `list books|highlights`
      search.ts         # `search`
      tag.ts            # `tag add|rm|list`
      favorite.ts       # `favorite add|rm`
      export.ts         # `export markdown`
      serve.ts          # `serve`
      db.ts             # `db path|info`
    output.ts           # table/JSON printers, stderr conventions
    errors.ts           # AppError -> exit code mapping
  core/
    paths.ts            # data dir resolution
    db/
      connection.ts     # open/create DB, PRAGMAs
      migrations.ts     # migration runner
      migrations/001-initial.ts
    repo/
      books.ts
      highlights.ts
      tags.ts
      importRuns.ts
    search.ts           # FTS5 + LIKE-fallback query builder
    export/
      markdown.ts       # rendering
      files.ts          # path safety, file naming
    time.ts             # ISO timestamp helpers, Apple epoch conversion
    ids.ts              # ULID
  sources/
    appleBooks/
      locate.ts         # find AEAnnotation/BKLibrary files
      open.ts           # copy-to-temp, read-only open
      validate.ts       # runtime schema validation
      extract.ts        # SELECTs + row mapping to SourceHighlight
      types.ts          # SourceBook / SourceHighlight DTOs
  import/
    pipeline.ts         # merge/dedup/upsert/archive, dry-run, run report
  server/
    app.ts              # buildServer(): fastify instance (no listen)
    security.ts         # host/origin checks, headers (see §11)
    routes/
      books.ts
      highlights.ts
      search.ts
      tags.ts
      imports.ts
      health.ts
web/                    # Vite + React SPA (source)
```

Dependency direction: `cli` and `server` depend on `core`, `sources`, `import`.
`core` depends on nothing above it. `sources/*` never touches quotable's own DB; it
only produces DTOs consumed by `import/pipeline.ts`.

## 3. Storage layout & configuration

### 3.1 Data directory

Resolution order (first hit wins):

1. `--data-dir <path>` global CLI flag
2. `QUOTABLE_DATA_DIR` environment variable
3. Default: `~/Library/Application Support/quotable` (macOS convention; on other
   platforms fall back to `$XDG_DATA_HOME/quotable` or `~/.local/share/quotable`)

Contents:

| Path | Purpose |
|---|---|
| `<data-dir>/quotable.db` | Canonical SQLite database |
| `<data-dir>/quotable.db-wal`, `-shm` | SQLite WAL sidecars |

The directory is created on first use with mode `0700`. The DB file must end up with
mode `0600`. No other config files in v1 — all configuration is flags/env.

### 3.2 SQLite settings

On every connection: `journal_mode=WAL`, `foreign_keys=ON`, `busy_timeout=5000`,
`synchronous=NORMAL`. One writer at a time is assumed (single user); the server holds
one connection.

## 4. Data model (canonical schema, migration 001)

Timestamps are ISO 8601 UTC strings (`YYYY-MM-DDTHH:mm:ss.sssZ`). IDs are ULIDs.

```sql
CREATE TABLE schema_migrations (
  version    INTEGER PRIMARY KEY,
  applied_at TEXT NOT NULL
);

CREATE TABLE books (
  id              TEXT PRIMARY KEY,
  source          TEXT NOT NULL CHECK (source IN ('apple_books')),
  source_asset_id TEXT NOT NULL,
  title           TEXT NOT NULL,
  author          TEXT,
  genre           TEXT,
  language        TEXT,
  created_at      TEXT NOT NULL,
  updated_at      TEXT NOT NULL,
  UNIQUE (source, source_asset_id)
);

CREATE TABLE highlights (
  id                 TEXT PRIMARY KEY,
  book_id            TEXT NOT NULL REFERENCES books(id),
  source             TEXT NOT NULL CHECK (source IN ('apple_books')),
  source_uuid        TEXT,             -- ZANNOTATIONUUID when present
  dedup_key          TEXT NOT NULL,    -- see §6.5
  text               TEXT NOT NULL,
  note               TEXT,
  chapter            TEXT,
  location           TEXT,             -- epubcfi string
  location_start     REAL,             -- numeric in-book ordering
  style              INTEGER,          -- raw source style value
  favorite           INTEGER NOT NULL DEFAULT 0 CHECK (favorite IN (0,1)),
  archived           INTEGER NOT NULL DEFAULT 0 CHECK (archived IN (0,1)),
  highlighted_at     TEXT,             -- source creation date
  source_modified_at TEXT,
  imported_at        TEXT NOT NULL,
  updated_at         TEXT NOT NULL,
  UNIQUE (source, dedup_key)
);
CREATE INDEX idx_highlights_book     ON highlights(book_id, location_start);
CREATE INDEX idx_highlights_archived ON highlights(archived);

CREATE TABLE tags (
  id         TEXT PRIMARY KEY,
  name       TEXT NOT NULL,            -- display form as first entered
  name_norm  TEXT NOT NULL UNIQUE,     -- normalize: NFKC, trim, toLowerCase
  created_at TEXT NOT NULL
);

CREATE TABLE highlight_tags (
  highlight_id TEXT NOT NULL REFERENCES highlights(id) ON DELETE CASCADE,
  tag_id       TEXT NOT NULL REFERENCES tags(id)       ON DELETE CASCADE,
  created_at   TEXT NOT NULL,
  PRIMARY KEY (highlight_id, tag_id)
);

CREATE TABLE import_runs (
  id          TEXT PRIMARY KEY,
  source      TEXT NOT NULL,
  started_at  TEXT NOT NULL,
  finished_at TEXT,
  status      TEXT NOT NULL CHECK (status IN ('running','success','failed')),
  dry_run     INTEGER NOT NULL DEFAULT 0 CHECK (dry_run IN (0,1)),
  stats_json  TEXT,                    -- ImportStats JSON, see §6.6
  error       TEXT
);

CREATE VIRTUAL TABLE highlights_fts USING fts5(
  text, note,
  content='highlights',
  content_rowid='rowid',
  tokenize='trigram'
);
-- Sync triggers (external-content pattern):
CREATE TRIGGER highlights_ai AFTER INSERT ON highlights BEGIN
  INSERT INTO highlights_fts(rowid, text, note) VALUES (new.rowid, new.text, new.note);
END;
CREATE TRIGGER highlights_ad AFTER DELETE ON highlights BEGIN
  INSERT INTO highlights_fts(highlights_fts, rowid, text, note)
  VALUES ('delete', old.rowid, old.text, old.note);
END;
CREATE TRIGGER highlights_au AFTER UPDATE OF text, note ON highlights BEGIN
  INSERT INTO highlights_fts(highlights_fts, rowid, text, note)
  VALUES ('delete', old.rowid, old.text, old.note);
  INSERT INTO highlights_fts(rowid, text, note) VALUES (new.rowid, new.text, new.note);
END;
```

Design notes:

- `archived` mirrors "deleted in the source app". Archived highlights are excluded from
  all default listings/search/export but retrievable with an explicit
  `--archived` / `archived=true` filter. Nothing is ever hard-deleted by import.
- `style` stores the raw integer; presentation-layer mapping to color names lives in
  one shared constant (`core/` §6.4) so a wrong guess is a one-line fix.
- The FTS `trigram` tokenizer is chosen for CJK substring search (ADR-001 §Japanese
  search). Queries shorter than 3 code points fall back to `LIKE` (§7 search rules).

## 5. Domain rules

- **Book identity**: `(source, source_asset_id)`. Title/author update on re-import if
  the source changed them.
- **Highlight identity**: `(source, dedup_key)` (§6.5).
- **quotable-owned fields** (never touched by import): `favorite`, tag links.
- **Source-owned fields** (overwritten by import when source is newer): `text`, `note`,
  `chapter`, `location`, `location_start`, `style`, `source_modified_at`.
- **Tag normalization**: display name = NFKC-normalized, trimmed, internal whitespace
  collapsed to single spaces; `name_norm` = display name lowercased. Rejected
  (`INVALID_TAG`): empty after trim, > 64 code points, names containing `,`, `#`, or
  control characters.
- **Orphan annotations** (book deleted in Books.app): imported under a synthetic book
  with `title = 'Unknown book (<assetId>)'`, `author = NULL`.

## 6. Apple Books importer

### 6.1 Locator (`sources/appleBooks/locate.ts`)

- Container root: `~/Library/Containers/com.apple.iBooksX/Data/Documents/`.
- Annotation DB: files in `AEAnnotation/` matching prefix `AEAnnotation` + suffix
  `.sqlite`; Library DB: files in `BKLibrary/` matching prefix `BKLibrary` + suffix
  `.sqlite`. Matching is implemented with `fs.readdirSync` + basename filtering (no
  glob dependency, per the §11.6 dependency budget). If multiple match, pick most
  recent mtime.
- `--db-dir <path>` on the import command overrides the container root (points at a
  directory with the same `AEAnnotation/`, `BKLibrary/` structure — enables importing
  from backups). This path comes from the user and is trusted like any local CLI input,
  but is still validated to exist and be a directory.
- Failure taxonomy (each maps to a distinct error, §12): container missing, permission
  denied (Full Disk Access), annotation DB missing, library DB missing (non-fatal:
  proceed with annotations only, all books become "Unknown book" — warn loudly).

### 6.2 Safe opening (`sources/appleBooks/open.ts`)

Never open Books.app's live files directly for reading:

1. Create a temp dir under `os.tmpdir()` with mode `0700`.
2. Copy `<db>`, `<db>-wal`, `<db>-shm` (the latter two if present) for both databases.
3. Open the copies with `better-sqlite3` in `readonly: true` mode.
4. `ATTACH` the BKLibrary copy to the annotation connection as `lib` (read-only), so
   extraction can use a single SQL join.
5. Always remove the temp dir afterwards (try/finally).

Rationale: avoids WAL locking interference with a running Books.app and guarantees
quotable can never corrupt the source (defense in depth on top of read-only mode).

### 6.3 Schema validation (`sources/appleBooks/validate.ts`)

Before extraction, verify via `PRAGMA table_info` that every table/column the
extraction query references exists (list defined in one constant, kept in sync with
`extract.ts` by a unit test). On mismatch, fail with error code
`APPLE_BOOKS_SCHEMA_MISMATCH` and a diagnostic listing missing tables/columns and the
found schema — this is the guardrail against silent Apple schema drift.

### 6.4 Extraction (`sources/appleBooks/extract.ts`)

Runs the reference query from `docs/research/apple-books-schema.md` (LEFT JOIN to
`lib.ZBKLIBRARYASSET`, extended with `b.ZGENRE` and `b.ZLANGUAGE`), with filters
applied in SQL:

- include rows where `ZANNOTATIONSELECTEDTEXT` is non-NULL and non-blank after trim;
- include **deleted** rows too (`ZANNOTATIONDELETED = 1`) — the pipeline needs them to
  archive local copies.

Mapping to `SourceHighlight` DTO:

| DTO field | Source | Transform |
|---|---|---|
| `uuid` | `ZANNOTATIONUUID` | as-is (nullable) |
| `assetId` | `ZANNOTATIONASSETID` | required; row skipped+counted if NULL |
| `text` | `ZANNOTATIONSELECTEDTEXT` | trim |
| `note` | `ZANNOTATIONNOTE` | trim; empty → null |
| `chapter` | `ZFUTUREPROOFING5` | trim; empty → null |
| `location` | `ZANNOTATIONLOCATION` | as-is |
| `locationStart` | `ZPLLOCATIONRANGESTART` | as-is (nullable REAL) |
| `style` | `ZANNOTATIONSTYLE` | as-is integer |
| `deletedAtSource` | `ZANNOTATIONDELETED` | `=== 1` |
| `createdAt` | `ZANNOTATIONCREATIONDATE` | Apple epoch → ISO (`+978307200`s); NULL-safe |
| `modifiedAt` | `ZANNOTATIONMODIFICATIONDATE` | same |
| `book.title` / `book.author` / `book.genre` / `book.language` | `lib` columns | NULL title → orphan rule §5 |

Style→name display mapping (`APPLE_STYLE_NAMES`, marked "unverified, from research"):
`0: underline, 1: green, 2: blue, 3: yellow, 4: pink, 5: purple`, unknown →
`unknown(<n>)`. The constant lives in `src/core/highlightStyle.ts` (core, not
`sources/`, so export and CLI can use it without violating the dependency
direction). Server responses include the derived `styleName` so the web UI never
duplicates the name mapping; the UI maps `styleName` → CSS color locally
(presentation only).

### 6.5 Dedup key

`dedup_key = uuid` when `uuid` is a non-empty string; otherwise
`'h:' + sha256hex(assetId + '\n' + (location ?? '') + '\n' + text)`.
The `'h:'` prefix prevents collisions between the two namespaces.

### 6.6 Merge pipeline (`import/pipeline.ts`)

Runs inside a single SQLite transaction (dry-run uses the same code path but rolls
back). For each extracted row:

```
resolve book:      upsert by (source, asset_id); update title/author/genre/language if changed
existing = SELECT by (source, dedup_key)
if !existing && !deletedAtSource  -> INSERT highlight            [added]
if !existing && deletedAtSource   -> skip                        [skippedDeleted]
if existing && deletedAtSource    -> set archived=1 if not set   [archived]
if existing && !deletedAtSource:
    if source modifiedAt > existing.source_modified_at (or existing is NULL)
                                  -> UPDATE source-owned fields, archived=0  [updated]
    elif both modifiedAt NULL AND content differs (text/note/chapter/style/location)
                                  -> UPDATE source-owned fields, archived=0  [updated]
    else                          -> no-op                       [unchanged]
```

- Highlights present locally but absent from the current source snapshot are left
  untouched (a source DB from a backup may be partial).
- `ImportStats` JSON: `{ booksAdded, booksUpdated, added, updated, archived, unchanged,
  skippedDeleted, skippedInvalid }`.
- Every run (including dry runs and failures) writes an `import_runs` row; failure
  stores the error message. The stats row is the run report shown by CLI/API.

### 6.7 Import state machine

`import_runs.status`: `running → success | failed`. A `running` row older than 1 hour
found at startup of a new run is marked `failed` with error `stale` (crash recovery;
no locking needed beyond SQLite's).

## 7. Search behavior

Implemented once in `core/search.ts`, used by CLI and API identically:

- Input: raw user query string, filters (bookId, tag, favorite, archived), pagination.
- If query length >= 3 Unicode code points: FTS5 `MATCH` where the user string is
  embedded as a **single quoted phrase** with internal `"` doubled — user input is
  never interpreted as FTS query syntax (no `AND`/`OR`/`*` operators in v1). Escaping
  is centralized and tested against hostile inputs (§11.4).
- Else: `WHERE (text LIKE '%'||?||'%' ESCAPE '\' OR note LIKE ...)` with `%`,`_`,`\`
  escaped in the parameter.
- Results ordered by book title then `location_start`; response includes a snippet.
  FTS path: `snippet(highlights_fts, 0, '«', '»', '…', 20)`; if the returned snippet
  contains no `«` marker (match was in `note`), use `snippet(..., 1, ...)` instead.
  LIKE path: first 160 code points of `text` with `«»` inserted around the first
  match.
- Default filter excludes `archived=1`.

## 8. CLI design

### 8.1 Conventions

- Binary name: `quotable` (`bin` in package.json → `dist/cli/index.js`).
- Global flags: `--data-dir <path>`, `--json`, `--quiet`, `--version`, `--help`.
- Human output → stdout tables (plain aligned text, no color dependency); `--json`
  emits a single JSON document to stdout. Logs/warnings → stderr.
- Exit codes: `0` success · `1` unexpected error · `2` usage error ·
  `3` source not found · `4` permission denied (Full Disk Access guidance printed) ·
  `5` source schema mismatch.
- Every command's `--json` output shape is defined in its issue and is a stable
  contract (tests assert it). Exception: `serve` is long-running and rejects
  `--json` with a usage error.

### 8.2 Commands

| Command | Behavior |
|---|---|
| `quotable import apple-books [--db-dir <path>] [--dry-run]` | §6; prints run report |
| `quotable list books [--sort title\|recent]` | id, title, author, highlight count |
| `quotable list highlights [--book <id>] [--tag <name>] [--favorites] [--archived] [--limit N] [--offset N]` | filtered listing |
| `quotable search <query> [--book <id>] [--tag <name>] [--favorites] [--limit N] [--offset N]` | §7 |
| `quotable tag add <highlight-id> <name...>` / `tag rm <highlight-id> <name...>` / `tag list` | tag management |
| `quotable favorite add <highlight-id>` / `favorite rm <highlight-id>` | favorite flag |
| `quotable export markdown --out <dir> [--book <id>] [--include-archived]` | §9 |
| `quotable serve [--port <n>] [--open]` | §10; default port 8484 |
| `quotable db path` / `db info` | data dir path; counts + schema version |

Highlight IDs are ULIDs; all ID-taking commands accept unambiguous ID prefixes (>= 6
chars) and error listing candidates when ambiguous.

## 9. Markdown export

- One file per book into `--out <dir>`: `<sanitized-title>--<book-id>.md`. Sanitization:
  strip control chars, replace `/:\\` and other filesystem-reserved chars with `-`,
  collapse whitespace, cap at 100 code points; book-id suffix guarantees uniqueness.
- Path safety: `--out` is resolved to an absolute path; every produced file path is
  verified (after resolution) to be inside it; export refuses to follow a `--out` that
  resolves into the quotable data dir.
- Full-overwrite semantics: export rewrites the files it generates deterministically
  (stable ordering by `location_start`); it never deletes other files in the directory.
- Fixed built-in template (custom templates are v2):

```markdown
---
title: "{{title}}"
author: "{{author}}"
source: apple_books
quotable_book_id: {{book_id}}
exported_at: {{iso timestamp}}
highlights: {{count}}
---

# {{title}}

## Highlights

> {{text}}

- Note: {{note}}          (omitted when null)
- Chapter: {{chapter}}    (omitted when null)
- Tags: #tag1 #tag2       (omitted when empty)
- Highlighted: {{highlighted_at date}} · Style: {{color name}}
```

- YAML string values use double-quote style with backslash escaping: `\` → `\\`,
  `"` → `\"`, CR/LF → single space, other control characters stripped; highlight
  text is emitted as blockquote
  lines with `> ` prefix per line (multiline-safe). Archived highlights excluded unless
  `--include-archived`.

## 10. HTTP server & Web UI

### 10.1 Server

- `buildServer(deps)` returns a Fastify app (listen is separate → testable with
  `inject`); a companion `startServer(app, port)` helper owns `listen` and always
  binds `127.0.0.1`.
- Binds `127.0.0.1` only. Default port `8484`; `--port` overrides; refuses `0.0.0.0`.
- Serves `dist/web/` at `/` (SPA fallback to `index.html` for non-`/api` GETs);
  JSON API under `/api`.

### 10.2 API endpoints

All responses JSON; errors use `{ "error": { "code": string, "message": string } }`.
The paginated list endpoint `/api/highlights` returns
`{ items, total, limit, offset, mode }` (default `limit=50`, max 200; `mode` is
`'list'` or `'search'` depending on the `q` parameter). `/api/books` returns
`{ items, total }`, `/api/tags` returns `{ items }`, and `/api/imports` returns
`{ items }` (recent runs, `limit` query param only) — unpaginated in v1 (bounded by
a personal library's size).

| Method & path | Purpose | Notes |
|---|---|---|
| `GET /api/health` | liveness | `{ status: "ok", version }` |
| `GET /api/books?sort=title\|recent` | book list | includes `highlightCount` |
| `GET /api/books/:id` | book detail | 404 code `NOT_FOUND` |
| `GET /api/highlights?bookId&tag&favorite&archived&q&limit&offset` | filtered list | `q` uses §7; each item embeds `tags: string[]` and book title/author |
| `PATCH /api/highlights/:id` | body `{ favorite: boolean }` | only quotable-owned fields |
| `PUT /api/highlights/:id/tags` | body `{ tags: string[] }` | replaces tag set; creates unknown tags |
| `GET /api/tags` | tag list with usage counts | |
| `POST /api/imports` | body `{ source: "apple_books", dryRun?: boolean }` | synchronous; returns run report; 409 `IMPORT_RUNNING` if one is active |
| `GET /api/imports?limit` | recent run reports | |

Validation via Fastify JSON schemas on every route (unknown body/query fields
rejected). Mutating routes require `Content-Type: application/json`.

### 10.3 Web UI (React SPA)

Routes:

| Route | View |
|---|---|
| `/` | Library: book cards (title, author, count), sort toggle, "Import from Apple Books" button with result toast |
| `/books/:id` | Book detail: highlight list ordered by position; favorite star toggle; tag chips editable inline (add/remove); archived hidden behind a toggle |
| `/search?q=` | Global search box + results with snippets, filter by tag/favorite |
| `/tags` | Tag list with counts → click filters highlights |
| `/favorites` | Favorite highlights across books |

Constraints: no external network requests (all assets bundled; verified by CSP §11.3);
fetch wrapper handles the API error envelope; state via plain fetch + React hooks
(no state-management or data-fetching dependency in v1).

## 11. Security model

quotable is local-only, single-user, no secrets stored. The security goal:
**running `quotable serve` must not give web pages running in the user's browser, or
malformed source files, any way to read or modify the user's data.** Other local OS
users are out of scope for the network boundary in v1 (accepted risk, ADR-004; token
auth is a v2 option) and are mitigated at rest by the §3.1 file permissions.

### 11.1 Trust boundaries

| # | Boundary | Untrusted side | Requirements |
|---|---|---|---|
| B1 | Browser ↔ HTTP server | Any web page running in the user's browser | §11.3 |
| B2 | Apple Books DBs → importer | Apple-controlled, possibly corrupt/malicious file content | §11.4 |
| B3 | CLI args/env → app | Local user (trusted) but validated for safety | path validation §9, §6.1 |
| B4 | DB/user text → Web UI DOM | Highlight text, notes, tag names, book titles | §11.5 |
| B5 | Export → filesystem | File writes derived from untrusted titles | §9 path safety |
| B6 | Supply chain | npm dependencies, CI | §11.6 |

### 11.2 Threat model summary

In scope: DNS-rebinding and CSRF against the localhost server; XSS via stored
highlight/book text; path traversal via book titles or `--out`; FTS/SQL injection via
search input; SQLite parsing of attacker-influenced files; a second local OS user
reading the data dir. Out of scope: an attacker with the user's own OS account, Apple
Books itself being compromised, physical access.

### 11.3 HTTP boundary (B1) — hard requirements

1. Bind `127.0.0.1` only, never configurable to other interfaces in v1.
2. **Host allowlist**: reject requests (400 `BAD_HOST`) unless `Host` is exactly
   `127.0.0.1:<port>`, `localhost:<port>`, or `[::1]:<port>` — blocks DNS rebinding.
3. **Origin check**: for state-changing methods (POST/PATCH/PUT/DELETE), if an `Origin`
   header is present it must be `http://127.0.0.1:<port>`, `http://localhost:<port>`,
   or `http://[::1]:<port>` (mirrors the Host allowlist); otherwise 403 `BAD_ORIGIN`.
   GETs must remain side-effect free.
4. No CORS headers ever (deny cross-origin reads by default).
5. Response headers on everything: `Content-Security-Policy: default-src 'self';
   img-src 'self' data:; style-src 'self'; script-src 'self'; connect-src 'self';
   base-uri 'none'; form-action 'none'; frame-ancestors 'none'`,
   `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer`; `/api` responses
   also `Cache-Control: no-store`.
6. Mutating routes require `Content-Type: application/json` (415 otherwise) — combined
   with 3, blocks HTML-form CSRF.
7. Request body limit 1 MiB; JSON schema validation rejects unknown fields.

These are acceptance criteria of issue 20 and tested by issue 28.

### 11.4 Parser/file boundaries (B2)

- Apple DBs opened read-only, from a private temp copy (§6.2), with
  `PRAGMA trusted_schema=OFF` executed on every connection to them (including the
  one-time checkpoint connections) — plus: no user-defined functions, no `ATTACH` of
  non-quotable-controlled paths beyond the two known copies, never `PRAGMA
  writable_schema`.
- All values read from Apple DBs are treated as untrusted strings/numbers: length-capped
  (text/note 64 KiB, others 4 KiB), type-checked in the row mapper; oversized/invalid
  rows are skipped and counted in `skippedInvalid`, never crash the run.
- Search input is never concatenated into SQL — always bound parameters; FTS phrase
  escaping per §7 with hostile-input tests (`"`, `*`, `(`, `NEAR`, emoji, mixed CJK).

### 11.5 Web UI output (B4)

React's default escaping everywhere; `dangerouslySetInnerHTML` is forbidden (ESLint
rule `react/no-danger` set to error). FTS snippets use marker strings converted to
React elements, never HTML injection.

### 11.6 Supply chain & release (B6)

- Runtime dependency budget: the packages named in §2.1 only; adding a runtime dep
  requires an ADR note.
- `package-lock.json` committed; CI runs `npm ci` and `npm audit --omit=dev`
  (fail on high/critical).
- GitHub Actions pinned to major versions; no secrets required by CI in v1.
- `.npmignore`/`files` allowlist so only `dist/`, `README`, `LICENSE` are published.
- Data dir `0700` / DB `0600` (§3.1) protects against other local users.

## 12. Error handling

Single `AppError` class in `core` with `code` (stable string), `message` (one line,
actionable), optional `details`. CLI maps codes → exit codes (§8.1) and prints
remediation hints for: `FULL_DISK_ACCESS_REQUIRED` (System Settings path),
`APPLE_BOOKS_NOT_FOUND`, `APPLE_BOOKS_SCHEMA_MISMATCH` (asks user to file an issue with
the printed diagnostic), `PORT_IN_USE`. The API maps the same codes to HTTP statuses.
Unexpected errors: generic one-line message; the stack is printed only when the
`QUOTABLE_DEBUG=1` environment variable is set (CLI) or in the server log.

## 13. Testing & validation strategy

| Layer | Approach |
|---|---|
| core/repo/search | vitest unit tests against real in-memory/temp SQLite |
| Apple Books source | fixture generator (issue 09) builds synthetic AEAnnotation/BKLibrary SQLite files matching the researched schema, incl. edge cases (deleted rows, NULL uuid, orphan asset, CJK text, emoji, 64KiB text) |
| import pipeline | scenario tests: fresh import, idempotent re-run, source update, source delete → archive, tag/favorite preservation |
| CLI | run built CLI as child process against temp data dir + fixtures; assert stdout JSON and exit codes |
| API | `fastify.inject` including all §11.3 security cases |
| Web UI | `npm run build` must pass in CI; manual checklist in issue 27 (no browser E2E in v1) |
| Whole-product smoke | issue 27: scripted end-to-end (import fixtures → search → tag → export → serve → curl API) |

CI (GitHub Actions): lint + typecheck + unit/integration tests on
`ubuntu-latest` (Node 20, 22) and `macos-latest` (Node 22); `npm audit` gate; build
artifacts (`dist/`) produced and smoke-tested.

## 14. Packaging & release

- npm package `quotable` (if the name is taken on npm, fall back to
  `@saber5656/quotable` — decision recorded at publish time), MIT license, README with
  install/usage, `engines.node >= 20`.
- `prepack` runs full build (backend `tsc` + web `vite build` into `dist/web`).
- v1 release = tag `v1.0.0` + npm publish (manual; publishing automation is v2).

# Research: Apple Books Annotation Storage on macOS

Status: researched 2026-07-08 (web sources; NOT yet verified against a live local database — see "Verification status" below).

This document records what is known about how Apple Books (Books.app) stores
highlights/annotations on macOS, and the assumptions quotable's importer is built on.
It is the design input for the Apple Books importer issues.

## Verification status

- The development machine used for this research has an empty
  `~/Library/Containers/com.apple.iBooksX/Data/Documents/` (no AEAnnotation/BKLibrary
  directories), so the schema below is compiled from external sources only.
- **Implementation MUST NOT hard-code this schema blindly.** A schema-validation layer
  (see DESIGN.md §6.3 and issue 11) checks the actual tables/columns at runtime and fails
  with a diagnostic report when they drift.
- First implementation task touching the importer must re-verify against a real database
  populated by Books.app (open Books.app, highlight something in any free book, then
  inspect the files below).

## Database locations

Two Core Data SQLite databases inside the Books.app sandbox container:

| Purpose | Directory | File name pattern |
|---|---|---|
| Annotations (highlights, notes) | `~/Library/Containers/com.apple.iBooksX/Data/Documents/AEAnnotation/` | `AEAnnotation_v10312011_1727_local.sqlite` (version-stamped; treat as glob `AEAnnotation*.sqlite`) |
| Library metadata (books) | `~/Library/Containers/com.apple.iBooksX/Data/Documents/BKLibrary/` | `BKLibrary-1-091020131601.sqlite` (version-stamped; treat as glob `BKLibrary*.sqlite`) |

Notes:

- File names embed a schema-version stamp and have been stable for years, but the
  importer must locate them by glob, not exact name. If multiple files match, pick the
  most recently modified.
- SQLite sidecar files (`-wal`, `-shm`) may be present; the importer copies all of
  `{db, db-wal, db-shm}` to a temp directory and opens the copy read-only (see
  DESIGN.md §6.2) to avoid locking or corrupting Books.app's live database.
- On modern macOS, terminal processes may need **Full Disk Access** to read this
  container. Attempting to read without it yields `EPERM`/"operation not permitted".
  The importer must detect this and print remediation steps rather than a raw stack
  trace.

## Annotation table: `ZAEANNOTATION`

Relevant columns (Core Data naming; all prefixed `Z`):

| Column | Meaning |
|---|---|
| `Z_PK` | Integer primary key |
| `ZANNOTATIONASSETID` | Book identifier; joins to `ZBKLIBRARYASSET.ZASSETID` |
| `ZANNOTATIONUUID` | UUID string for the annotation (stable dedup key) |
| `ZANNOTATIONSELECTEDTEXT` | The highlighted text (NULL for bookmarks) |
| `ZANNOTATIONNOTE` | User's note attached to the highlight (nullable) |
| `ZANNOTATIONREPRESENTATIVETEXT` | Surrounding/representative text (nullable) |
| `ZFUTUREPROOFING5` | Chapter/section title (nullable; despite the name) |
| `ZANNOTATIONTYPE` | Annotation kind (bookmark vs highlight/note; highlights observed as type 2) |
| `ZANNOTATIONSTYLE` | Highlight color/style: 0=underline, 1=green, 2=blue, 3=yellow, 4=pink, 5=purple (verify at implementation) |
| `ZANNOTATIONISUNDERLINE` | Underline flag (present in newer schemas; nullable) |
| `ZANNOTATIONDELETED` | Soft-delete flag; 1 = deleted in Books.app, must be filtered/mapped |
| `ZANNOTATIONLOCATION` | Position, typically an `epubcfi(...)` string |
| `ZPLLOCATIONRANGESTART` | Numeric position used for in-book ordering |
| `ZANNOTATIONCREATIONDATE` | Creation timestamp (Core Data epoch, REAL) |
| `ZANNOTATIONMODIFICATIONDATE` | Last modification timestamp (Core Data epoch, REAL) |

### Timestamp format

Core Data timestamps are seconds since **2001-01-01T00:00:00Z** (Apple epoch), stored as
REAL. Conversion: `unix_seconds = value + 978307200`. Values may have fractional
seconds; may be NULL.

### Row-filtering rules for import

- Import rows where `ZANNOTATIONSELECTEDTEXT IS NOT NULL AND TRIM(...) <> ''`
  (bookmarks and empty selections are excluded).
- `ZANNOTATIONDELETED = 1` rows are surfaced as "deleted at source" so quotable can
  archive (not hard-delete) the corresponding local highlight.
- `ZANNOTATIONUUID` is the primary dedup key. Fallback identity when NULL:
  `sha256(assetId + '\n' + location + '\n' + selectedText)`.

## Library table: `ZBKLIBRARYASSET`

Relevant columns:

| Column | Meaning |
|---|---|
| `ZASSETID` | Book identifier (join key) |
| `ZTITLE` | Book title |
| `ZAUTHOR` | Author display string |
| `ZGENRE` | Genre (nullable) |
| `ZLANGUAGE` | Language code (nullable) |
| `ZPATH` | Local file path of the book (nullable) |
| `ZCONTENTTYPE` | Content kind (EPUB vs PDF vs audiobook; exact values to verify) |
| `ZISFINISHED` | Reading-finished flag |
| `ZLASTOPENDATE` | Last opened (Core Data epoch) |

### Orphan annotations

Deleting a book in Books.app removes its `ZBKLIBRARYASSET` row but **keeps** its
annotations. The importer must therefore LEFT JOIN and handle NULL titles: such
highlights import under a synthetic book titled `Unknown book (<assetId>)` with
`author = NULL`.

## Reference query (shape only; final query defined in issue 12)

```sql
SELECT
  a.ZANNOTATIONUUID              AS uuid,
  a.ZANNOTATIONASSETID           AS asset_id,
  a.ZANNOTATIONSELECTEDTEXT      AS selected_text,
  a.ZANNOTATIONNOTE              AS note,
  a.ZFUTUREPROOFING5             AS chapter,
  a.ZANNOTATIONSTYLE             AS style,
  a.ZANNOTATIONDELETED           AS deleted,
  a.ZANNOTATIONLOCATION          AS location,
  a.ZPLLOCATIONRANGESTART        AS location_start,
  a.ZANNOTATIONCREATIONDATE      AS created_at_apple,
  a.ZANNOTATIONMODIFICATIONDATE  AS modified_at_apple,
  b.ZTITLE                       AS title,
  b.ZAUTHOR                      AS author,
  b.ZGENRE                       AS genre,
  b.ZLANGUAGE                    AS language
FROM ZAEANNOTATION a
LEFT JOIN lib.ZBKLIBRARYASSET b ON a.ZANNOTATIONASSETID = b.ZASSETID
ORDER BY a.ZANNOTATIONASSETID, a.ZPLLOCATIONRANGESTART;
```

(`lib` = the BKLibrary database ATTACHed to the annotation connection, or two separate
connections with an application-level join — decided in DESIGN.md §6.2.)

## Known limitations

- **PDF annotations** are stored differently and are out of scope for v1 (documented
  non-goal).
- Schema is undocumented and can change with any macOS/Books update; hence the
  validation layer and the copy-then-read strategy.
- iCloud sync state can differ between devices; quotable only reads what the local Mac's
  Books.app has synced locally.

## Sources

- [I built a CLI to break my highlights out of Apple Books (dev.to, Andrey Korchak)](https://dev.to/andreykorchak/i-built-a-cli-to-break-my-highlights-out-of-apple-books-51jn)
- [Extracting notes and highlights from iBooks (Kevin Cunningham)](https://kevincunningham.co.uk/posts/extracting-notes-and-highlights-from-ibooks/)
- [Exporting Book Metadata from Apple Books (Ryne Andal)](https://ryneandal.github.io/2026/05/07/exporting-book-metadata-from-apple-books)
- [annotations-from-ibooks (GitHub)](https://github.com/bf4648/annotations-from-ibooks)
- [obsidian-ibook-plugin issue #59 — PDF annotations not exportable](https://github.com/bingryan/obsidian-ibook-plugin/issues/59)

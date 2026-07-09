# Title

CLI command: `export markdown`

## Summary

Implement `quotable export markdown --out <dir> [--book <id>] [--include-archived]`
over the export module.

## Context

Thin adapter over issue 18 (DESIGN.md §8.2, §9). The help text must state the
overwrite semantics (ADR-002: exported files are generated artifacts).

## Scope

- `src/cli/commands/export.ts` + child-process tests.

## Detailed Requirements

1. `--out <dir>` required (missing → exit 2). `--book` accepts ID/≥6-char prefix.
   `--include-archived` passes through.
2. Calls `exportBooks`; human output: one line per file
   (`wrote <absolute path> (<n> highlights)`) then a summary line
   (`Exported N books to <absolute outDir>`). `--json`:
   `{ outDir, files: [{ path, bookId, highlights }] }` — `outDir` and every `path`
   are absolute resolved paths (tests assert this).
3. Command help includes: "Exported files are regenerated on each export; manual
   edits will be overwritten."
4. Empty database / no matching book: `Nothing to export.` (stdout), exit 0.
5. `INVALID_OUT_DIR`, `PATH_ESCAPE`, unknown book → mapped errors (exit 1 or 2 per
   issue 05 conventions; `NOT_FOUND` → 1, usage → 2).

## Acceptance Criteria

- Child-process test: seed 2 books → export → files exist with expected names and
  golden content; re-export after tagging a highlight updates the file.
- `--book` prefix filter exports one file; unknown book errors.
- `--out` pointing into the data dir → error, exit 1, nothing written.
- `--json` shape exact.

## Validation

- `npm test`.

## Dependencies

- 05, 18

## Non-goals

- Custom templates, watch mode, deleting removed books' old files (documented v2).

## Design References

- DESIGN.md §8.2, §9; ADR-002

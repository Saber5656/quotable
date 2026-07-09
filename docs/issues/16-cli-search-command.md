# Title

CLI command: `search`

## Summary

Implement `quotable search <query> [--book <id>] [--tag <name>] [--favorites]
[--limit N] [--offset N]` on top of the shared search module.

## Context

Thin CLI adapter over `core/search.ts` (issue 04); conventions from issue 05.

## Scope

- `src/cli/commands/search.ts` + child-process tests.

## Detailed Requirements

1. Positional `<query>` required; empty/whitespace → exit 2 with usage.
2. Flags map to `SearchFilters` (`--book` with prefix resolution like issue 15;
   `--favorites` → favorite=true). No `--archived` flag: search never returns
   archived rows in v1 CLI.
3. Table columns: `ID` (8), `BOOK` (≤30), `SNIPPET` (≤70, `«»` markers as-is),
   `TAGS`, `FAV`.
4. `--json`: `SearchResult` from issue 04 verbatim
   (`{ items, total, limit, offset }`).
5. Zero hits: `No matches.` to stdout, exit 0.
6. Errors from the search module (`EMPTY_QUERY`) → exit 2.

## Acceptance Criteria

- CJK 2-char query (LIKE path) and ≥3-char query (FTS path) both return the seeded
  row via child-process tests.
- Hostile query strings from issue 04's list run through the CLI with exit 0 and
  sane output.
- Filter combination test (`--book` + `--favorites`).
- JSON snapshot stable.

## Validation

- `npm test`.

## Dependencies

- 04, 05, 06

## Non-goals

- Query syntax (boolean operators), relevance sorting, archived search.

## Design References

- DESIGN.md §7, §8.2

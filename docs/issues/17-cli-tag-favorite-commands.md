# Title

CLI commands: `tag` and `favorite`

## Summary

Implement `quotable tag add|rm|list` and `quotable favorite add|rm` over the tag and
highlight repositories.

## Context

quotable-owned metadata management from the terminal (DESIGN.md §8.2). Repos from
issues 06/07; ID-prefix rules shared with other commands.

## Scope

- `src/cli/commands/tag.ts`, `src/cli/commands/favorite.ts` + child-process tests.

## Detailed Requirements

1. `tag add <highlight-id> <name...>`: resolves highlight by ID/prefix; adds one or
   more tags (`addTags`); prints resulting tag list (`tags: a, b, c`).
   `--json`: `{ highlightId, tags: string[] }`.
2. `tag rm <highlight-id> <name...>`: `removeTags`; same outputs. Removing an
   unassigned tag is a silent no-op (idempotent UX).
3. `tag list`: table `TAG`, `COUNT` (issue 07 ordering);
   `--json`: `{ items: [{ id, name, count }] }`.
4. `favorite add <highlight-id>` / `favorite rm <highlight-id>`: `setFavorite`;
   prints `favorited <id8>` / `unfavorited <id8>`;
   `--json`: `{ highlightId, favorite: boolean }`. Idempotent.
5. Error mapping: unknown highlight → exit 1 with `NOT_FOUND` message; ambiguous
   prefix → exit 2 listing candidates; `INVALID_TAG` → exit 2 with the reason.
6. Multi-tag adds are transactional (all-or-nothing when any name is invalid).

## Acceptance Criteria

- Add/rm/list round-trip via child-process tests; NFKC unification observable from
  the CLI: add `Ｂｏｏｋ` to one highlight and `Book` to another, `tag list` shows
  exactly one tag row (display name = first spelling) with count 2.
- Favorite toggle reflected in `list highlights --favorites`.
- Invalid tag name (`a#b`, 65-char, empty) → exit 2; nothing persisted from the same
  invocation.
- All `--json` shapes exact.

## Validation

- `npm test`.

## Dependencies

- 05, 07 (06 transitively)

## Non-goals

- Tag rename/merge/delete (v2), bulk operations, favorites listing command
  (covered by `list highlights --favorites`).

## Design References

- DESIGN.md §5, §8.2

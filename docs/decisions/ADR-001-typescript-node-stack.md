# ADR-001: TypeScript/Node single-package stack

Status: Accepted (2026-07-08, confirmed with product owner)

## Context

quotable v1 is a CLI plus a localhost web UI backed by SQLite. Implementation will be
delegated to lower-capability coding agents executing granular issues, so the stack
must maximize mechanical executability and ecosystem familiarity. Candidates: Go
(single binary, weaker UI story), Python (packaging friction), Rust (highest
implementation difficulty), TypeScript/Node.

## Decision

TypeScript 5.x (`strict`), ESM, Node >= 20, one npm package. `commander` for the CLI,
`better-sqlite3` for storage, `fastify` for HTTP, React 18 + Vite for the web UI
(built into `dist/web`, served statically). ULIDs for IDs, vitest for tests. No npm
workspaces; `web/` is a Vite root inside the single package.

Runtime dependency budget is fixed to the packages named in DESIGN.md §2.1; adding a
runtime dependency requires updating this ADR or a new one.

## Consequences

- One language across CLI, server, and UI; issues stay uniform and small.
- npm distribution (`npm i -g quotable`) instead of a single static binary; Node >= 20
  becomes a user prerequisite. Acceptable for the developer-adjacent target audience.
- `better-sqlite3` is a native module: install requires prebuilt binaries (available
  for macOS arm64/x64) — CI must exercise macOS.
- The FTS5 `trigram` tokenizer bundled with better-sqlite3's SQLite provides CJK
  substring search without extra dependencies (DESIGN.md §7); queries under 3 code
  points use a LIKE fallback.

## Alternatives rejected

- **Go**: best distribution story, but the bundled-SPA toolchain and typed shared
  models across API/UI are weaker; UI issues would grow.
- **Python**: parser-friendly but global CLI installation and one-package web bundling
  are fragile for end users.
- **Rust**: quality ceiling highest, execution risk for delegated granular issues also
  highest.

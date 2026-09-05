# Title

Project scaffolding, toolchain, and CI

## Summary

Initialize the quotable npm package: TypeScript (strict, ESM), directory layout,
lint/format/test toolchain, and a GitHub Actions CI workflow. After this issue the
repo builds, lints, and runs an empty test suite in CI.

## Context

quotable is a single npm package containing a CLI, an HTTP server, and a Vite-built
web UI (DESIGN.md §2). Every later issue assumes this toolchain exists. Stack choices
are fixed by ADR-001; do not substitute libraries.

## Scope

- `package.json`, `tsconfig.json`, ESLint + Prettier config, vitest config
- `src/` and `web/` directory skeletons (empty placeholder modules only)
- `.github/workflows/ci.yml`
- `.gitignore`, `LICENSE` (MIT), minimal README stub

## Detailed Requirements

1. `package.json`:
   - `"name": "quotable"`, `"private": false`, `"type": "module"`,
     `"engines": { "node": ">=20" }`, `"license": "MIT"`.
   - `bin`: `{ "quotable": "./dist/cli/index.js" }` (file may be a stub for now).
   - Scripts: `build` (backend `tsc -p tsconfig.build.json` + `vite build` for `web/`
     into `dist/web`), `typecheck`, `lint`, `format:check`, `test` (vitest run),
     `dev:web` (vite dev server).
   - Runtime deps (exact set, ADR-001): `commander`, `better-sqlite3`, `fastify`,
     `@fastify/static`, `ulid`. Dev deps (exact list): `typescript`, `vitest`,
     `vite`, `@vitejs/plugin-react`, `react`, `react-dom`, `react-router-dom`
     (React packages are devDependencies — Vite bundles them into `dist/web`; they
     are not runtime deps of the Node package), `@types/node`,
     `@types/better-sqlite3`, `@types/react`, `@types/react-dom`, `eslint`,
     `typescript-eslint`, `eslint-plugin-react`, `prettier`,
     `@testing-library/react`, `jsdom` (for web component tests in issue 24).
   - Commit `package-lock.json`.
2. `tsconfig.json`: `strict: true`, `module`/`moduleResolution` `NodeNext`, `target`
   `ES2022`, `noUncheckedIndexedAccess: true`. Separate `tsconfig.build.json` compiling
   `src/` → `dist/`, excluding tests and `web/`.
3. ESLint: typescript-eslint recommended-type-checked; rule `react/no-danger: error`
   for `web/`; Prettier as formatter (no conflicting rules).
4. Directory skeleton exactly as DESIGN.md §2.3 (create directories with placeholder
   `index.ts` or `.gitkeep`; no logic).
5. `web/`: minimal Vite + React app (`web/index.html`, `web/src/main.tsx` rendering
   "quotable"), `vite.config.ts` with `build.outDir = '../dist/web'`, `emptyOutDir`.
6. CI workflow `ci.yml`:
   - Triggers: `push` to `main`, `pull_request`.
   - Matrix: `ubuntu-latest` × Node `20`, `22`; plus `macos-latest` × Node `22`.
   - Steps: `actions/checkout@v4`, `actions/setup-node@v4` (with npm cache),
     `npm ci`, `npm run lint`, `npm run typecheck`, `npm run build`, `npm test`,
     `npm audit --omit=dev --audit-level=high`. Build runs **before** test because
     later issues' CLI tests spawn the built `dist/cli/index.js`.
7. `.gitignore`: `node_modules/`, `dist/`, `*.log`, `.DS_Store`, coverage output.
8. `LICENSE`: MIT, copyright holder "quotable contributors".

## Acceptance Criteria

- `npm ci && npm run build && npm test && npm run lint && npm run typecheck` all exit 0
  locally on macOS.
- CI workflow passes on a PR containing this change.
- `node dist/cli/index.js` runs without throwing (stub is fine).
- Repo contains no generated files under version control (`dist/` ignored).

## Validation

- Run the five commands above; attach output to the PR.
- Verify `npm pack --dry-run` lists only intended files (add `files: ["dist"]`).

## Dependencies

None (first issue).

## Non-goals

- Any product logic, DB code, or real CLI commands.
- Publishing to npm (issue 29).

## Design References

- DESIGN.md §2 (stack, module map), §11.6 (supply chain), §13 (CI)
- ADR-001

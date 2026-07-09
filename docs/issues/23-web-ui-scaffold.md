# Title

Web UI scaffold and build integration

## Summary

Stand up the React SPA shell in `web/`: routing, layout, API client wrapper, shared
UI primitives, and the Vite build wired into `dist/web` so the server can serve it.
Feature views land in issues 24/25.

## Context

DESIGN.md §10.3. Constraints: no external network resources (CSP `self`), no
`dangerouslySetInnerHTML`, plain CSS, plain hooks (no react-query).

## Scope

- `web/src/` application shell + `npm run build` integration; a Vite dev-proxy for
  local development.

## Detailed Requirements

1. Routing (`react-router-dom`, `BrowserRouter`): routes `/`, `/books/:id`,
   `/search`, `/tags`, `/favorites` rendering placeholder components; header nav
   with links to Library, Search, Tags, Favorites; active-link styling.
2. API client `web/src/api.ts`:
   - `apiFetch<T>(path, init?): Promise<T>` — base `/api`, sets
     `Content-Type: application/json` on bodies, parses the error envelope and
     throws `ApiError { code, message, status }`.
   - Typed functions for every endpoint of issues 21/22 (types mirrored in
     `web/src/types.ts`; keep in sync manually — no codegen in v1).
3. Shared primitives (`web/src/components/`): `Spinner`, `ErrorBox` (renders
   `ApiError.message` as text), `Toast` (success/error, auto-dismiss 5s),
   `EmptyState`. All render user data as text nodes only.
4. Plain CSS: one `app.css` with CSS variables (light theme only), system font
   stack; no CSS framework, no icon font, no web fonts.
5. Vite config: `build.outDir = '../dist/web'`; dev server proxies `/api` to
   `http://127.0.0.1:8484` (`npm run dev:web` against a running `quotable serve`).
6. `npm run build` produces `dist/web/index.html` + hashed assets; all asset URLs
   relative/`self`-hosted (verify no `http(s)://` references in build output).
7. ESLint `react/no-danger: error` active for `web/` (from issue 01; verify).

## Acceptance Criteria

- `npm run build` emits the SPA into `dist/web`; serving it via issue 20's static
  layer renders the shell and navigation works (manual check + one inject test that
  `/` returns HTML).
- `grep -R "dangerouslySetInnerHTML" web/src` → empty; build output contains no
  external URLs (scripted check in CI or test).
- `apiFetch` unit-tested (happy path + envelope error) with a mocked `fetch`.
- Placeholder pages reachable at all five routes in the dev server.

## Validation

- `npm test`, `npm run build`, manual dev-server walkthrough.

## Dependencies

- 01, 20 (the acceptance criteria exercise issue 20's static serving)

## Non-goals

- Real data views (24/25), dark mode, i18n (v1 UI text is English).

## Design References

- DESIGN.md §10.3, §11.5; ADR-001

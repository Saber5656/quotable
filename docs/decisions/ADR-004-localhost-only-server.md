# ADR-004: Localhost-only, unauthenticated web server with browser-boundary hardening

Status: Accepted (2026-07-08, confirmed with product owner)

## Context

`quotable serve` hosts the web UI and JSON API. The product is single-user and
local-only; the data (reading highlights) is low-sensitivity but private. The main
realistic attackers are **web pages in the user's own browser** (CSRF, DNS rebinding
against localhost services) rather than remote network peers. Options: token
authentication, localhost binding with hardening, or no protections.

## Decision

- Bind `127.0.0.1` only; binding other interfaces is not configurable in v1.
- No authentication.
- Mandatory browser-boundary hardening (DESIGN.md §11.3): strict `Host` allowlist
  (DNS-rebinding defense), `Origin` validation on all state-changing methods,
  `Content-Type: application/json` requirement on mutations, no CORS headers, strict
  CSP, `X-Content-Type-Options: nosniff`, `Cache-Control: no-store` on `/api`,
  side-effect-free GETs.

These controls are acceptance criteria (issue 20) with dedicated tests (issue 28), not
best-effort extras.

## Consequences

- Zero-friction UX (`quotable serve` → open browser), no secret storage or token
  handling code.
- A second OS user on the same machine could access the port. Accepted for v1 given
  data sensitivity; data-at-rest is still protected by `0700`/`0600` file modes. Token
  auth can be added in v2 behind a flag without breaking the API shape.
- Malicious web pages cannot read (no CORS, SOP) or mutate (Origin + Content-Type
  checks) quotable data, and DNS rebinding is neutralized by the Host allowlist.

## Alternatives rejected

- **Startup token in URL** (`?token=...`): stronger multi-user posture but worse UX,
  token leakage via browser history, and more code; deferred to v2 as an option.
- **No hardening**: unacceptable — localhost services are a well-known CSRF/rebinding
  target.

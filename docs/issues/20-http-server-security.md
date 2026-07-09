# Title

HTTP server skeleton and security middleware

## Summary

Implement `src/server/app.ts` (`buildServer`) and `src/server/security.ts`: a Fastify
instance with the complete DESIGN.md §11.3 browser-boundary hardening, the error
envelope, static SPA serving, and `GET /api/health`. Feature routes land in issues
21/22.

## Context

ADR-004 makes these controls mandatory acceptance criteria, not extras. Every later
route inherits them, so they must exist before any endpoint ships. Issue 28 attacks
this surface.

## Scope

- `app.ts`, `security.ts`, `routes/health.ts` + `fastify.inject` tests.

## Detailed Requirements

1. `buildServer(deps: { db: Database; port: number; version: string;
   webRoot: string | null }): FastifyInstance` — no `.listen` inside; the `port` is
   needed for Host/Origin allowlists. A companion export
   `startServer(app: FastifyInstance, port: number): Promise<void>` owns `listen`
   and always passes `host: '127.0.0.1'` — binding is enforced here and nowhere
   else (issue 26 calls this helper).
2. `security.ts` implements (registered as global hooks):
   - **Host allowlist** (`onRequest`): `Host` must be exactly `127.0.0.1:<port>`,
     `localhost:<port>`, or `[::1]:<port>` → else 400
     `{ error: { code: 'BAD_HOST', ... } }`.
   - **Origin check** (`onRequest`, methods POST/PATCH/PUT/DELETE): if `Origin`
     present it must be `http://127.0.0.1:<port>`, `http://localhost:<port>`, or
     `http://[::1]:<port>` (mirrors the Host allowlist) → else 403 `BAD_ORIGIN`.
   - **Content-Type gate**: mutating methods require
     `Content-Type: application/json` (parameters allowed) → else 415
     `BAD_CONTENT_TYPE`.
   - **Response headers** (`onSend`, all responses): CSP exactly per DESIGN.md §11.3
     item 5, `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer`;
     additionally `Cache-Control: no-store` for any `/api/*` URL.
   - No CORS plugin, no `Access-Control-*` headers anywhere.
3. Body limit 1 MiB (`bodyLimit`). `ajv` config: `removeAdditional: false` +
   schemas use `additionalProperties: false` (unknown fields → 400).
4. Error envelope: all errors serialize to `{ error: { code, message } }`;
   Fastify validation errors → 400 `VALIDATION`; unknown routes under `/api` → 404
   `NOT_FOUND`; `AppError` maps code→status via a table
   (`NOT_FOUND`→404, `IMPORT_RUNNING`→409, `VALIDATION`/`INVALID_*`→400, default 500
   with generic message — internal details never leaked).
5. Static serving: when `webRoot` non-null, `@fastify/static` at `/`; GET/HEAD of
   unknown non-`/api` paths → `index.html` (SPA fallback). API routes win over
   static.
6. `routes/health.ts`: `GET /api/health` → `{ status: 'ok', version }`.
7. GETs must be side-effect free (convention documented in the module header;
   enforced by review + issue 28).

## Acceptance Criteria

- inject-tests: good Host passes; `Host: evil.com:8484`, `Host: 127.0.0.1:9999`,
  missing Host → 400 `BAD_HOST`.
- POST with `Origin: https://evil.com` → 403; no Origin → passes Origin check (CLI
  curl case); correct Origin → passes.
- POST with `Content-Type: text/plain` and form content types → 415.
- Every response (200, 400, 404) carries CSP/nosniff/referrer headers; `/api/*` has
  `no-store`.
- No response anywhere contains `Access-Control-Allow-Origin`.
- Unknown `/api/x` → 404 envelope; `/some/spa/route` → index.html when webRoot set.
- Body > 1 MiB → 413.

## Validation

- `npm test` (`fastify.inject`; no real sockets needed except one `startServer`
  smoke test asserting the OS-reported bind address is `127.0.0.1`).

## Dependencies

- 01, 03

## Non-goals

- Feature endpoints (21, 22), the SPA itself (23), CLI `serve` (26), auth (v2).

## Design References

- DESIGN.md §10.1, §10.2 (envelope), §11.3, §12; ADR-004

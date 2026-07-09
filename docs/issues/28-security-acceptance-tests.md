# Title

Security acceptance test suite

## Summary

A dedicated adversarial test suite verifying every DESIGN.md §11.3/§11.4 requirement
end-to-end: DNS rebinding, CSRF vectors, header posture, injection resistance, path
traversal, and file permissions. Release (issue 29) is gated on this suite.

## Context

Issue 20 implements the controls and tests them unit-style; this issue attacks the
assembled server and CLI the way a hostile web page or file would, and pins the
posture against regressions (ADR-004; ISSUE_PLAN.md §6 "Security gate").

## Scope

- `test/security/*.test.ts` (inject + real-socket where noted). No production code
  changes expected; found gaps become fixes in the owning modules with tests here.

## Detailed Requirements

Test groups (each case asserts both status/behavior and the error envelope):

1. **DNS rebinding**: requests with `Host` = attacker domains (with/without port,
   IP-lookalikes `127.0.0.1.evil.com`, `localhost.evil.com`) → 400; correct hosts
   with wrong port → 400.
2. **CSRF**: cross-origin POST/PATCH/PUT with `Origin: https://evil.com` → 403;
   form-encoded and `text/plain` bodies (form-CSRF shapes) → 415 even with a good
   Origin; GET requests proven side-effect free (favorite count unchanged after
   GET-flood of every read route).
3. **Header posture** (every route × 200/400/404): exact CSP string of DESIGN.md
   §11.3, `X-Content-Type-Options`, `Referrer-Policy`, `/api` `Cache-Control:
   no-store`, absence of any `Access-Control-*` header.
   Also: request body > 1 MiB → 413; unknown body/query fields on every schema'd
   route → 400.
4. **Injection**: FTS/SQL hostile corpus (issue 04 list + `"; ATTACH DATABASE`,
   long-input 100 KiB query → 400 by schema maxLength or handled) via
   `GET /api/highlights?q=`; tag names attempting SQL/HTML; all return controlled
   errors or safe results.
5. **Stored XSS corpus**: import fixture rows containing `<script>`, `<img onerror>`,
   markdown-breaking text; assert API returns them as JSON strings (no HTML content
   type) and the snippet marker contract holds (`«»` only from the server, never
   raw user HTML interpretation — regression test at the API layer).
6. **Path traversal**: export with book titles `../../x`, `..\\x`, `a/b`, NUL bytes
   (via direct repo insert) → files stay inside `--out` (assert real resolved
   paths); `--out` = data dir → refused.
7. **Local file posture** (POSIX): after `db info` on a fresh dir: data dir `0700`,
   DB `0600`; Apple source files' mtime/bytes unchanged after an import run.
   Additionally, malicious-source hardening: a fixture Apple DB with a hostile
   schema addition (e.g., a view or trigger referencing forbidden functions) still
   imports or fails safely — `PRAGMA trusted_schema` is verified OFF on the
   extraction connection and no write to any Apple DB copy source occurs.
8. **Server reachability**: with the server listening, a connection attempt to the
   machine's LAN IP on the same port is refused (real-socket test; skip when no
   non-loopback interface).
9. Suite is table-driven so new attack strings can be appended without structural
   changes; each group references the DESIGN.md §11 clause it verifies in a comment.

## Acceptance Criteria

- All groups implemented and green in CI (ubuntu + macos jobs; group 8 may be
  macos-only if needed).
- Any discovered failure is fixed in the owning module within this issue (or a
  blocking follow-up issue is filed and linked) — the suite must end green.
- CI marks this suite required for the release issue (29 references it).

## Validation

- `npm test` (suite included in default run); CI green.

## Dependencies

- 13, 18, 19, 20, 21, 22 (import pipeline and export are exercised by the
  injection/traversal/file-posture groups)

## Non-goals

- Penetration testing of Node/Fastify themselves, fuzzing (v2 nice-to-have), auth.

## Design References

- DESIGN.md §11 (all subsections), §12; ADR-004

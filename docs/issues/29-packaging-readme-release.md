# Title

Packaging, README, and release readiness

## Summary

Finish the npm package for public release: `files` allowlist, prepack build, README,
CHANGELOG, npm-publish dry run, and the pre-release checklist (security gate, history
scan).

## Context

DESIGN.md §14 and §11.6; the repository is public OSS under MIT. Publishing itself is
manual by the maintainer; this issue makes it a one-command act.

## Scope

- `package.json` finishing touches, `README.md`, `CHANGELOG.md`, `.github/` docs
  files; release checklist executed and recorded.

## Detailed Requirements

1. `package.json`: `files: ["dist", "README.md", "LICENSE"]`; `prepack` runs the
   full build; `description`, `keywords` (`apple-books`, `highlights`, `readwise`,
   `obsidian`, `sqlite`), `repository`, `bugs`, `homepage` fields set.
2. `npm pack` tarball inspected in CI: must contain `dist/cli/index.js` (executable
   shebang), `dist/web/index.html`; must NOT contain `src/`, `test/`, fixtures, maps
   over 5 MB total (size budget check: tarball < 15 MB).
3. README sections: what/why (local Readwise for Apple Books), install
   (`npm i -g quotable`, Node >= 20, macOS for import), quickstart (import → serve),
   every CLI command with an example, Full Disk Access setup with a screenshot-free
   text walkthrough, exported-Markdown-is-generated warning (ADR-002), data location
   & backup note, security posture summary (ADR-004), development
   (build/test/`test:e2e`), license.
4. `CHANGELOG.md` (Keep a Changelog format) with `1.0.0` entry.
5. Package-name decision (known unknown U5): first check availability with
   `npm view quotable name` (E404 = available; otherwise taken). If taken, switch
   to `@saber5656/quotable` (update README/package, add `publishConfig.access:
   "public"`) and record the decision in ADR-001 as an amendment. Then run
   `npm publish --dry-run` as the final publish rehearsal and attach its output.
6. Pre-release checklist (recorded in the issue):
   - CI green including e2e (27) and security suite (28);
   - live Apple Books verification from issue 14 completed with evidence linked,
     or an explicit maintainer waiver recorded here (ISSUE_PLAN.md §6 release
     gate);
   - `npm audit --omit=dev` no high/critical;
   - GitHub Actions workflows audited: all `uses:` pinned to major versions, no
     repository secrets referenced (DESIGN.md §11.6);
   - git history scanned for secrets/PII: run `gitleaks detect --source .
     --no-banner --redact` (install via `brew install gitleaks`, record the
     version) and attach the clean report output as evidence;
   - LICENSE year/holder correct; no personal absolute paths in docs or code
     (`grep -R "/Users/" src web docs` clean, excluding legitimate examples with
     `$HOME`-style placeholders).
7. Tag `v1.0.0` created only after the checklist passes (tagging may be done by the
   maintainer; document the command).

## Acceptance Criteria

- `npm pack` contents and size budget verified in CI.
- README covers every shipped command (cross-checked against `quotable --help`
  output).
- Dry-run publish succeeds; name decision recorded.
- Checklist executed with evidence linked in the issue/PR.

## Validation

- CI job for pack-inspection; manual checklist evidence.

## Dependencies

- All of 01–28 (release gate).

## Non-goals

- Actual `npm publish` (maintainer action), Homebrew formula, binary builds (v2),
  publish automation/provenance (v2).

## Design References

- DESIGN.md §11.6, §14; ADR-001, ADR-002, ADR-004; ISSUE_PLAN.md §8 U5

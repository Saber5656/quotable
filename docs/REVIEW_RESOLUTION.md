# Review resolution contract

This addendum is a documentation-only acceptance contract for PR #1. It records the resolution required for each existing review thread. It does not claim that product implementation or tests have been completed. The existing Bot review is the sole Bot input for this PR and will not be retriggered.

## PRRT_kwDOTNkErc6PuzYw — proxy rewrites the backend Host

Finding: A Vite proxy without changeOrigin can forward the browser Host instead of the backend host expected by the service on port 8484.

Normative resolution:
- The development proxy must set changeOrigin: true and rewrite the forwarded Host to the backend origin on port 8484.
- The route mapping, path rewrite, and origin policy must be documented together; a browser-controlled Host must not select an unintended backend.
- Production deployment must not silently depend on the development proxy behavior.

Focused verification before resolving this thread:
- Send a proxied request with a browser Host and assert the backend receives the expected 8484 origin/Host.
- Test the configured route and a non-matching route, including an invalid backend target, and assert deterministic failure.
- Confirm production configuration has an explicit equivalent or explicitly does not use the dev proxy.

## PRRT_kwDOTNkErc6PuzYx — import failure is recorded before dependent stages

Finding: Starting locate/open/validation/extraction before recording the import run can lose pre-pipeline failure evidence.

Normative resolution:
- Start an import run record before locate, open, schema validation, or extraction.
- Record pre-pipeline failures (missing database, permission/open failure, unsupported source) against that run with a terminal status and diagnostic category.
- Downstream stages must not create a success-like record when the source cannot be opened.

Focused verification before resolving this thread:
- Inject locate and open failures and assert an import run exists with a pre-pipeline failure status before the error is returned.
- Verify no locate/open/validation/extraction success event is emitted after the pre-pipeline failure.
- Repeat the failure and retry path to confirm each run is independently traceable and idempotent.

## PRRT_kwDOTNkErc6PuzYz — importer exposes an async concurrency seam

Finding: A synchronous import seam cannot exercise concurrent second POST behavior or prove the 409 mutex contract.

Normative resolution:
- The importer must expose an asynchronous Promise<ImportReport> seam that can be paused at a deterministic stage.
- The HTTP handler must serialize concurrent imports with a mutex/lock and return 409 for a second request while the first is active.
- Release of the lock must be exception-safe, including pre-pipeline and validation failures.

Focused verification before resolving this thread:
- Start two concurrent POSTs against the paused importer and assert one owns the lock while the other receives 409.
- Release the first request through success and each failure path, then assert a later request can proceed.
- Confirm the async report preserves the documented import result and does not hide the mutex outcome.

## PRRT_kwDOTNkErc6PuzY3 — live source clears archived state

Finding: A live source must clear its archived flag even when no other source-owned fields changed.

Normative resolution:
- When the source is observed as live, the merge/upsert operation must set archived=false for that source-owned row unconditionally.
- This update must occur even when title/content/date and every other source-owned value compare equal.
- Rows owned by another source must not be unarchived as a side effect.

Focused verification before resolving this thread:
- Seed an archived live-owned row with otherwise identical source data and assert the import clears archived.
- Seed a row with no other changes and a row owned by another source and assert only the live-owned row changes.
- Verify repeat imports are idempotent.

## PRRT_kwDOTNkErc6PuzY6 — locationStart participates in content diff

Finding: When modifiedAt is null, comparing content alone misses a changed locationStart.

Normative resolution:
- The content/diff predicate must include locationStart whenever modifiedAt is null, with explicit null/number comparison semantics.
- A changed locationStart must produce the documented modified/update result even when text and modifiedAt are unchanged.
- The predicate must not invent a timestamp solely to make the comparison pass.

Focused verification before resolving this thread:
- Compare null-modifiedAt records with same text and different locationStart values and assert they are modified.
- Cover null-to-number, number-to-null, equal-number, and modifiedAt-present cases.
- Confirm the diff report identifies locationStart as the changed field.

## PRRT_kwDOTNkErc6PuzY8 — whitespace-only rows are skipped after JavaScript trim

Finding: Empty rows containing tabs or newlines are not empty under a space-only check.

Normative resolution:
- Apply JavaScript trim semantics to the normalized row text before deciding whether a highlight row is empty.
- Skip rows that become empty after trim, including spaces, tabs, newlines, and mixed whitespace.
- Preserve meaningful non-whitespace text and keep row numbering/reporting deterministic.

Focused verification before resolving this thread:
- Exercise empty strings, spaces, tabs, newlines, mixed Unicode/ASCII whitespace, and meaningful text.
- Assert whitespace-only rows do not create records or errors, while non-empty rows retain their original source location.
- Repeat the same input to confirm deterministic output.

## PRRT_kwDOTNkErc6PuzZB — usage errors exit with code 2

Finding: CLI usage/AppErrors must map to exit code 2, not the generic operational failure code 1.

Normative resolution:
- The CLI error mapper must classify usage/AppErrors as exit 2.
- Operational failures remain exit 1 and successful commands remain exit 0.
- The JSON/human error payload must retain the category so callers can distinguish invalid invocation from runtime failure.

Focused verification before resolving this thread:
- Exercise missing/invalid arguments and assert exit 2 with usage diagnostics.
- Exercise a valid invocation that encounters an operational failure and assert exit 1.
- Confirm success returns 0 and that all three modes agree on the category mapping.

## PRRT_kwDOTNkErc6PuzZD — output directory containment is checked before mkdir

Finding: Creating the output directory before validating containment can write to an invalid data-dir target.

Normative resolution:
- Resolve and validate output-directory containment under the approved data directory before any mkdir, file creation, or parent mutation.
- Reject absolute, traversal, symlink-escape, and equivalent normalized targets before filesystem writes.
- A failed containment check must leave the filesystem unchanged and report the rejected path safely.

Focused verification before resolving this thread:
- Test valid nested output, parent/equal target, traversal, absolute target, and symlink-escape cases.
- Snapshot the filesystem before invalid requests and assert no directory or file is created.
- Verify the containment check uses canonical paths and handles missing parents without creating them first.

## Scope and review boundary

This file is a design/acceptance contract only. It is not evidence that the implementation or focused checks have already passed. After the relevant implementation and validation evidence exists, each mapped existing thread may be replied to and resolved individually. No Bot review will be triggered again.
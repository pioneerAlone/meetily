# Spec 0001: Auto-Fallback GGUF Downloads

## Problem Statement

When a user installs Meetily on a machine with ≥14 GB of RAM, the onboarding
flow recommends the `qwen3.5:4b` GGUF summary model. Meetily downloads this
model from `huggingface.co` during onboarding. From networks where
`huggingface.co` is unreachable (rate-limited, blocked, or behind a firewall),
the download times out, and the user cannot complete onboarding or use the
local summary feature.

The user has no way to recover from this without manually downloading the GGUF
to the correct directory. The error message does not explain where to put the
file, what filename to use, or that a mirror exists.

## Solution

When the built-in GGUF download fails for a network or HTTP-error reason,
automatically retry the download against `hf-mirror.com` (a public mirror of
the same HuggingFace artifacts). The retry is transparent: the user sees one
continuous download progress, and the existing UI does not change.

If the fallback also fails, surface the original error from the primary source
plus a hint pointing at `docs/specs/0001-auto-fallback-gguf-downloads.md` (the
manual download path that ships with this change).

## User Stories

1. As a user behind a restricted network, I want the GGUF download to
   transparently retry against a mirror, so that I can complete onboarding
   without configuring anything.
2. As a user behind a restricted network, I want to see one continuous
   progress bar during primary + fallback attempts, so that I am not surprised
   by two distinct download phases.
3. As a user with both `huggingface.co` and `hf-mirror.com` reachable, I want
   the download to succeed from `huggingface.co` first, so that I get the
   canonical artifact.
4. As a user whose network blocks `huggingface.co` but not `hf-mirror.com`, I
   want the download to succeed after a single automatic fallback attempt, so
   that the onboarding flow completes in a reasonable time.
5. As a user whose network blocks both sources, I want a clear error message
   that names the primary and fallback URLs, so that I can manually download
   the file and place it in the models directory.
6. As a user who cancels a download mid-fallback, I want the cancellation to
   take effect immediately, so that the in-progress fallback stops and the
   partial file is cleaned up.
7. As a user upgrading from a previous Meetily version, I want my existing
   downloaded models to remain valid, so that I do not have to re-download.
8. As a developer reading the codebase, I want the source-resolution logic in
   one module with focused tests, so that adding new mirrors later is a single
   function change.
9. As a developer reading the codebase, I want the download manager to remain
   the single entry point for GGUF downloads, so that UI callers and partial
   download recovery do not need to change.
10. As a developer reading the codebase, I want the failure classification to
    be explicit (which errors trigger fallback, which do not), so that
    transient failures do not silently cascade.

## Implementation Decisions

- A new module `download_source` is added under
  `frontend/src-tauri/src/summary/summary_engine/` containing the source
  list, the host-swap resolver, and the failure classification.
- The `ModelDef` struct gains no new fields. The `download_url` field remains
  the primary source URL. The fallback URL is derived at runtime by host swap
  inside the resolver.
- The `ModelManager::download_model` method is refactored so the actual HTTP
  fetch is wrapped in a single inner function `download_once`, and the outer
  loop iterates over the ordered source list. The progress emission and
  partial-download recovery stay outside the inner function so they remain
  observable across source attempts.
- The fallback attempt does not preserve a partial file from the primary
  attempt. The fallback re-downloads the entire file. Rationale: the file is
  ≤2.6 GB today, the cost of resuming from a different source's byte range is
  not worth the complexity, and most fallback attempts will fully succeed.
- The cancellation flag is checked between sources, not just within one. If
  the user cancels during the primary attempt, the fallback is not attempted.
- The fallback source list is a hard-coded constant of length 2
  (`huggingface`, `hf-mirror`) in v1. Adding a third source in the future is a
  one-line change in this module.
- The `download_url` parser does not validate that it points at
  `huggingface.co`; if it does not, the model simply has no fallback. Today
  all four GGUF entries point at `huggingface.co`, so this is safe.
- The Tauri command surface, the database schema, the frontend components, and
  the progress events are **unchanged**. The fallback is internal to the Rust
  download manager.

## Testing Decisions

What makes a good test: it exercises the resolver on real `ModelDef`
instances and asserts the resulting URL, or it exercises the failure
classifier on real `reqwest::Error` shapes. Tests do not hit the network.

Prior art in the codebase:

- `summary/summary_engine/commands.rs` has unit tests for the
  `recommend_summary_model` function (`recommended_summary_model_*` tests).
  We follow the same style: `#[cfg(test)]` module at the bottom of the file
  with `#[test]` functions on small pure logic.

Modules to test:

- `download_source::resolve_primary_url(&ModelDef) -> Url`
- `download_source::resolve_fallback_url(&Url) -> Option<Url>`
- `download_source::should_fallback(&reqwest::Error) -> bool`
- `download_source::should_fallback_from_status(reqwest::StatusCode) -> bool`

Manual integration test on macOS M2:

1. Build the app with `pnpm run tauri:build:metal`.
2. Block `huggingface.co` at the DNS level (e.g. via
   `/etc/hosts` → 127.0.0.1).
3. Launch the app, complete onboarding to the model-download step.
4. Confirm the download progresses, hits the fallback, and completes.
5. Unblock `huggingface.co`, restart, repeat, and confirm the primary source
   is used (shorter download time, no fallback log line).
6. Block both sources, restart, confirm the error message names both URLs.

## Out of Scope

- Whisper model downloads (`whisper_engine.rs`). Onboarding does not pull
  these; they are user-initiated only, and the upstream `devtest` branch
  already contains hardening for that path.
- Parakeet model downloads (`parakeet_engine.rs`). Parakeet v3 is already
  served from a reachable CDN
  (`meetily.towardsgeneralintelligence.com`).
- A user-facing source picker in the UI. The fallback is automatic only.
- Persisting which source "won" the last time. Every download restarts at the
  primary.
- Mirrors other than `hf-mirror.com`. Adding ModelScope, Aliyun, or others is
  a future decision; the architecture supports it but the list is fixed at
  length 2 in v1.
- Re-trying the primary before falling back (e.g. "try primary 3 times, then
  fallback"). Each source gets exactly one attempt in v1.
- Changing the user-visible error when both sources fail beyond a one-line
  hint pointing at the spec / manual path.
- Any change to the upstream repo's `CONTRIBUTING.md` or `BUILDING.md`.

## Further Notes

- The HF→hf-mirror host swap is safe today because every GGUF entry uses
  `https://huggingface.co/<org>/<repo>/resolve/main/<file>`, and
  `hf-mirror.com` mirrors the same path layout. If upstream changes a GGUF
  URL to a different host, that entry loses its fallback silently. A unit
  test guards against the silent regression.
- The fallback path goes through the same code that already supports range
  requests, so resuming a partial primary attempt on the fallback host is
  technically possible but we deliberately do not do it (see implementation
  decision above).
- The spec is published as issue #1 on `pioneerAlone/meetily` once the dev
  build is verified end-to-end on this machine.

# Spec 0001 — Ticket Breakdown

This is a local draft of the ticket breakdown for spec 0001
(`docs/specs/0001-auto-fallback-gguf-downloads.md`). It will be published to
the GitHub issue tracker once the dev build is verified end-to-end on this
machine.

Tickets are vertical slices, each cutting a complete path through the change.
Blocking edges are explicit.

## T01 — Resolver + fallback source list (no behaviour yet)

**What to build:** A new module that exposes the source list (`huggingface`,
`hf-mirror`) and pure functions that map a `ModelDef` or a URL to the
corresponding resolved URL on a chosen source. Plus a failure-classifier
function that takes a `reqwest::Error` or `StatusCode` and returns whether
fallback should trigger.

**Blocked by:** None (can start immediately).

**Acceptance criteria:**
- [ ] `download_source::resolve_primary_url(model_def)` returns the same URL
      currently stored in `model_def.download_url`.
- [ ] `download_source::resolve_fallback_url(model_def)` returns
      `https://hf-mirror.com/...` for every entry in `ModelDef::get_available_models()`,
      or `None` if the primary URL is not a `huggingface.co` URL.
- [ ] `download_source::should_fallback_from_status(404)` returns `true`;
      `should_fallback_from_status(200)` returns `false`.
- [ ] Unit tests at the bottom of the module exercise all four entries in
      `ModelDef` and assert exact resolved URLs.

---

## T02 — Wrap download loop in source iteration (the actual behaviour)

**What to build:** Refactor `ModelManager::download_model` so the single-shot
HTTP fetch lives in an inner `download_once(source)` function, and the outer
loop iterates over the ordered source list. On a classified fallback error
from the primary, automatically retry the same model against the fallback
source. The progress event emission and partial-download recovery stay
observable across source attempts as one continuous progress bar to the user.

**Blocked by:** T01 (need the resolver and classifier).

**Acceptance criteria:**
- [ ] Downloading a model whose primary source returns 404 transparently
      retries against the fallback and succeeds (verified manually with a
      blocked primary host).
- [ ] Downloading a model whose primary source succeeds does not make a
      fallback request (verified by network log).
- [ ] The progress bar in the UI shows one continuous download across both
      attempts; the user does not see two distinct download phases.
- [ ] Cancelling during the primary attempt stops before any fallback
      attempt.
- [ ] Existing unit tests in `summary_engine` still pass.

---

## T03 — Better error message when both sources fail

**What to build:** When both the primary and fallback attempts fail, the
error surfaced to the UI names both URLs and points at the spec file
(`/Users/wangbo/.../docs/specs/0001-auto-fallback-gguf-downloads.md`) for the
manual download path.

**Blocked by:** T02 (the loop has to exist first).

**Acceptance criteria:**
- [ ] The error message contains the primary URL.
- [ ] The error message contains the fallback URL.
- [ ] The error message contains a hint pointing at the spec file.
- [ ] Manual test: block both hosts, attempt download, confirm error message.

---

## T04 — End-to-end verification + branch push

**What to build:** Run the full onboarding flow on this M2 Mac against a
blocked `huggingface.co` and confirm the model downloads successfully via
the fallback. Capture the resulting `.app` artifact path and the log lines
emitted during the fallback.

**Blocked by:** T02, T03.

**Acceptance criteria:**
- [ ] `pnpm run tauri:build:metal` produces a `.app` artifact.
- [ ] Onboarding completes with `huggingface.co` blocked via `/etc/hosts`.
- [ ] The `builtin-ai-download-progress` event emits a complete (100%)
      progress value for the affected model.
- [ ] The model file at `~/Library/Application Support/com.meetily.ai/models/summary/`
      is byte-identical to the same file on `huggingface.co` (sha256
      compared).
- [ ] Push the branch and the spec to the fork on GitHub.

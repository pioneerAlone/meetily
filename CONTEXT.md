# Meetily Auto-Fallback GGUF Downloads

Personal fork of `Zackriya-Solutions/meetily` (`pioneerAlone/meetily`). The goal
of this fork is to make built-in GGUF summary model downloads succeed in
restricted-network environments without user intervention.

On machines with ≥14 GB RAM the onboarding flow recommends `qwen3.5:4b`
(2.6 GB), whose `gguf_file` lives at
`huggingface.co/unsloth/Qwen3.5-4B-GGUF`. From networks where
`huggingface.co` is unreachable (or rate-limited), the onboarding download
times out, leaving users stuck. The fix: when a download attempt against the
primary source fails for a network or HTTP-error reason, transparently retry
against a fallback source (`hf-mirror.com`) that mirrors the same artifacts.

Scope of v1 is the **GGUF summary model path only** (`summary_engine`).
Transcription models (Whisper, Parakeet) are out of scope for v1: Parakeet v3
is already served from a reachable CDN
(`meetily.towardsgeneralintelligence.com`), and the Whisper download path has
recently received upstream hardening
(`fix(whisper): reject undersized model downloads`,
`fix(whisper): finalize cancellation safely`,
`fix(whisper): sync model download state`) merged into `devtest`.

## Language

**Download attempt**: one execution of the HTTP download for a single model
file. Retries on transient connection errors are part of the same attempt; a
source-switch event starts a new attempt against a different source.

_Avoid_: download call, fetch, request.

**Source fallback**: the automatic switch from the primary source
(`huggingface.co`) to the fallback source (`hf-mirror.com`) when a download
attempt fails for a network or HTTP-error reason. Triggered inside the model
manager's download loop, not exposed to the user.

_Avoid_: mirror switch, retry, fallback strategy.

**Fallback source**: one entry in the ordered list of base URLs the resolver
tries for a given GGUF model. Today the list is exactly two: `huggingface`
followed by `hf-mirror`. The fallback source is the second entry; the primary
is the first.

**GGUF download resolver**: the function that takes a `ModelDef` and a
`Source` and returns the concrete URL for that model on that source. Lives in
`frontend/src-tauri/src/summary/summary_engine/download_source.rs`.

**Model catalog entry**: a row in `ModelDef::get_available_models()` declaring
one downloadable GGUF. Carries `name`, `gguf_file`, `download_url` (the HF URL
that becomes the primary source), and the template pieces the resolver needs
to mint a fallback URL.

**Transcription model entry**: a `(name, base_url)` pair in
`parakeet_engine/parakeet_engine.rs` describing one ONNX bundle. Out of scope
for v1.

_Avoid_: model definition, model record (those are DB terms).

## Source resolution rules

- Primary source for every model is whatever URL `download_url` in `ModelDef`
  points to today (currently `huggingface.co`). We do not rewrite it.
- Fallback source is constructed by swapping the host of `download_url` from
  `huggingface.co` to `hf-mirror.com`, preserving path, org, repo, file.
- The resolver does **not** parse the HF URL structure to mint the fallback; it
  does a literal host swap, because all GGUF entries today use the same
  `huggingface.co/<org>/<repo>/resolve/main/<file>` shape.
- If `download_url` does not parse as an HF URL (different host), the model
  has no fallback; download only tries the primary once and surfaces the
  error.
- The attempt order is fixed and global, not per-model: primary first, fallback
  second. v1 does not persist which source "won" last time; the next download
  always starts at primary.

## Failure classification

A download attempt is considered failed for fallback purposes when:

- The HTTP request errors before a response is received (DNS, TCP, TLS,
  timeout). This is the most common case in restricted networks.
- The HTTP response is `4xx` or `5xx`. The server is telling us the file is
  gone or we are being denied.

A download attempt is **not** considered failed (no fallback) when:

- The response is `200 OK` but the byte count differs from `Content-Length`
  (the existing partial-download recovery in `model_manager.rs` handles this).
- The user cancels the download.
- A local filesystem error occurs writing to disk.

## Environment notes (for future reference)

Captured from upstream `docs/BUILDING.md` and this machine's actual install
state on 2026-08-29. Useful context if we ever need to revisit why a build
worked (or didn't).

- macOS 14.4.1, Apple Silicon (M2). `uname -m = arm64`.
- Xcode CommandLineTools 15.3 installed at
  `/Library/Developer/CommandLineTools`; SDK `MacOSX14.4.sdk` available.
- Rust 1.96.0 (above the project's `rust-version = "1.77"` floor).
- Node 25.2.1, pnpm 11.8.0.
- Homebrew provides `cmake 4.1.1`, `ffmpeg`, `llvm 21.1.0` (with `libomp` at
  `/opt/homebrew/Cellar/llvm/21.1.0/lib/libomp.dylib`). Upstream BUILDING.md
  does not pin cmake or LLVM versions; we are deliberately not pinning them
  in this fork and capturing the actual versions here instead.
- Upstream `CONTRIBUTING.md:7-11` requires feature branches be created from
  `devtest`, not `main`. `main` is at `0281737` (release/v0.4.0); `devtest`
  is 18 commits ahead and contains whisper-download hardening that this work
  benefits from automatically (we branch from `devtest` HEAD).

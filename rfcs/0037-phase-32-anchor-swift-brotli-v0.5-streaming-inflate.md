# RFC-0037 — Phase 32 anchor: swift-brotli v0.5 (streaming inflate)

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-17 |
| Resolution | Accepted 2026-05-17 — Phase 32 plans may begin authoring. Single existing package: swift-brotli v0.5. |

## Summary

Anchor Phase 32 on **swift-brotli v0.5** — additive non-breaking minor version bump adding `Brotli.Streaming.Decoder` (init/update/finish) for streaming RFC 7932 Brotli decompression. Completes codec-tier streaming-decode (with deflate + gzip + zlib done via Phase 30-31). Independent of deflate (Brotli is a separate algorithm).

Single-tranche; per-tranche calendar **deferred to brainstorm reality-check** per Gate 22 canonical pattern. Likely Shape A (buffering wrap) per Phase 30 precedent (~30-60 min wall-clock).

## Problem

[Gate 31 retrospective](../docs/gates/2026-05-17-gate-31-retrospective.md) item 5 requires "Phase 32 anchor decision recorded as an RFC" before Phase 32 plans begin. The retrospective surveyed ten candidate waves; this RFC formalizes the choice.

**Concrete gap:** swift-brotli v0.4 ships one-shot decode (`Brotli.decode(_:)`) only — no streaming-decode counterpart. Phase 30-31 closed the deflate/gzip/zlib streaming-decode story; brotli is the last codec without streaming-decode.

**Why now (Phase 32):**
1. Phase 30-31 completed streaming-decode for 3 of 4 codecs (deflate + gzip + zlib). Brotli is the last.
2. Brotli is independent of deflate (separate algorithm); Phase 32 unblocks Phase 33+ (content-encoding v0.7 streaming-decode wiring).
3. Buffering-wrap option (Phase 30 precedent) keeps scope modest (~30-60 min).
4. Audience continuity from Phase 22-31 codec streaming sweeps.

**Why not the alternatives (full survey in Gate 31 retro § Item 5):**
- **swift-content-encoding v0.7 streaming-decode wiring** — blocked on Phase 32.
- **swift-deflate v0.6 true memory-streaming inflate** — v0.5 just shipped; demand-driven.
- **brotli/deflate v0.6 window carry** — ratio improvement; not blocking.
- **swift-oauth2-client v0.4** — v0.3 just shipped; no settle time.
- **B3 + Jaeger, idna v0.3** — no demand.
- **RS-family JWT** — STILL BLOCKED on swift-crypto `_RSA` SPI.
- **Package rename** — premature.

## Proposal

### Anchor: swift-brotli v0.5 (single existing package, additive minor bump)

**swift-brotli v0.5** — add `Brotli.Streaming.Decoder` alongside v0.1's `Brotli.decode(_:)` one-shot + v0.3's `Brotli.Streaming.Encoder` + v0.4's `drain()`. v0.1-v0.4 APIs unchanged.

| Addition | Description |
|---|---|
| `Brotli.Streaming.Decoder` struct | Sendable value-type streaming decoder. Mirrors `Brotli.Streaming.Encoder` shape: init / update(_:) / finish() throws -> Bytes. |
| `init()` | No parameters. |
| `update(_ chunk: Bytes)` | Buffer compressed input. Empty/finished no-op. |
| `finish() throws(BrotliError) -> Bytes` | Run `Brotli.decode(_:)` one-shot on buffered input. Throws on malformed input + double-call. |
| Possibly new error case | `BrotliError.decoderFinished` (mirror deflate/gzip/zlib pattern) — TBD by brainstorm. |

### Brainstorm decision: Shape A (buffering wrap) vs Shape B (state-machine refactor)

Per Phase 30's brainstorm-empowered-by-RFC scope simplification precedent, brainstorm chooses between:

- **Shape A (recommended): buffering wrap.** Internally accumulates compressed input + runs Brotli.decode(_:) one-shot at finish. Honest-scope-under-limitation pattern at 5th instance. ~30-60 min.
- **Shape B (alternate): state-machine refactor.** Refactor `Brotli.Sources/Brotli/Decoder.swift` + supporting helpers (BitReader, MetaBlockHeader, etc.) for yield-and-resume. ~3-5 hours.

Phase 30's precedent for deflate v0.5 was Shape A. Brotli's Decoder is more complex than deflate's Inflater (metablock structure, multiple sub-decoders, larger state surface). Shape A is the likely choice unless brainstorm uncovers a tractable state-machine boundary.

### Test surface (~12-15 tests)

| Test category | Scope |
|---|---|
| Round-trip via v0.2 one-shot encoder | Encode "hello"/"hello world"/pangram/70 KiB via Brotli.compress → stream-decode → bytes match | 4-5 tests |
| Multi-chunk input | Split a one-shot-encoded output into 2/3/many chunks → stream-decode → bytes match | 2-3 tests |
| Quality coverage | Stream-decode outputs from .fastest / .default / .smallest quality encoders | 2-3 tests |
| Error cases | Truncated input throws; double-finish throws decoderFinished; update-after-finish no-op | 3 tests |
| Edge cases | Empty stream; single-byte payload; tiny 1-byte chunks | 2-3 tests |

Estimated 12-15 new tests. Total package tests post-32A: 124 v0.4 + ~13 = ~137.

### Acceptance criteria

- v0.5.0 ships with `Brotli.Streaming.Decoder`.
- v0.1-v0.4 APIs unchanged. `Brotli.decode(_:)` continues byte-for-byte. `Brotli.compress(_:)` + `Streaming.Encoder` + `drain()` unchanged.
- Stream-decoded output equals one-shot `Brotli.decode(_:)` output for all test cases.
- CI green on macOS + Linux, first try.
- DocC includes `### Streaming decompress (v0.5+)` topic group.
- CHANGELOG + README updated with streaming-decode example + honest-scope note (if Shape A).
- **DocC discipline:** cross-package symbol refs use single-backtick per first-line MEMORY note (re-emphasized after Phase 31A's CI break).

### Out of scope (deferred to v0.6+)

- True memory-streaming inflate (state-machine refactor of Decoder.swift).
- `reset()` for decoder reuse.
- Window-carry-aware streaming (currently the encoder side defers this too).

### Migration (v0.4 → v0.5)

**Additive only — non-breaking.** All v0.1-v0.4 APIs unchanged.

### Risk

**Medium per brainstorm-locked shape:**
- **Shape A (buffering wrap)**: Low risk. Pure wrapper around Brotli.decode. ~30-60 min.
- **Shape B (state-machine refactor)**: High risk. Brotli's Decoder is more complex than deflate's. ~3-5hr.

| Risk | Mitigation |
|---|---|
| Adopters expect memory-streaming and get buffering wrap | Document honestly in CHANGELOG (matches Phase 30 + 31 pattern). |
| Cross-package DocC ref slip | Re-emphasized per Phase 31A CI break. Single-backtick for cross-package refs. |

## Alternatives considered

See [Gate 31 retrospective](../docs/gates/2026-05-17-gate-31-retrospective.md) § Item 5 for the full candidate matrix.

## Dependencies

- No new direct dependencies.

## References

- [RFC 7932](https://www.rfc-editor.org/rfc/rfc7932) — Brotli compressed data format.
- [Gate 31 retrospective](../docs/gates/2026-05-17-gate-31-retrospective.md) — anchor decision rationale.
- [RFC-0027](0027-phase-22-anchor-swift-brotli-v0.3-streaming-encoder.md) — Phase 22 (brotli v0.3 streaming encoder); the canonical Streaming.Encoder shape.
- [RFC-0035](0035-phase-30-anchor-swift-deflate-v0.5-streaming-inflate.md) — Phase 30 (deflate v0.5 buffering-wrap streaming decode); the brainstorm-empowered-by-RFC Shape A precedent.
- [feedback_docc_cross_package](MEMORY first-line) — single-backtick for cross-package DocC refs.

## Decision

**Accepted 2026-05-17.** Phase 32 anchored on **swift-brotli v0.5 streaming inflate**. Brainstorm + plan + execute via the bare-swift inline-execution pattern. Per-tranche calendar estimate **deferred to brainstorm reality-check** per Gate 22's canonical pattern.

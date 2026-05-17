# RFC-0036 — Phase 31 anchor: swift-gzip v0.5 + swift-zlib v0.5 streaming inflate (2-tranche phase)

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-17 |
| Resolution | Accepted 2026-05-17 — Phase 31 plans may begin authoring. Two existing-package tranches: 31A swift-gzip v0.5, 31B swift-zlib v0.5. |

## Summary

Anchor Phase 31 on a **coordinated 2-tranche sweep** adding `Streaming.Decoder` to swift-gzip and swift-zlib. Both wrap `Deflate.Streaming.Decoder` (from Phase 30 v0.5) + decode their own header/trailer framing + verify checksums. Completes the deflate-family streaming-decode story (deflate + gzip + zlib all stream-capable on both encode and decode sides).

Per-tranche scope is small (~30-60 min, wrapper-pattern calibration). Total Phase 31 ~1-2 hours wall-clock.

## Problem

[Gate 30 retrospective](../docs/gates/2026-05-17-gate-30-retrospective.md) item 5 requires "Phase 31 anchor decision recorded as an RFC" before Phase 31 plans begin. The retrospective surveyed eleven candidate waves; this RFC formalizes the choice.

**Concrete gap:** swift-gzip v0.4 and swift-zlib v0.4 ship one-shot decoders (`Gzip.decode(_:)` and `Zlib.decode(_:)`) only. After Phase 30's `Deflate.Streaming.Decoder`, the deflate-family streaming-decode story is incomplete without wrapper streaming decoders for gzip and zlib.

**Why now (Phase 31):**
1. Phase 30 shipped deflate v0.5 streaming-decode. gzip + zlib wrap deflate; they're the natural downstream.
2. Wrapper-pattern, modest per-tranche scope (~30-60 min each).
3. Closes the deflate-family streaming-decode story before tackling brotli (Phase 32) and content-encoding wiring (Phase 33+).
4. Audience continuity from Phase 22-30 codec sweeps.

**Why not the alternatives (full survey in Gate 30 retro § Item 5):**
- **swift-brotli v0.5 streaming inflate** — independent of deflate; state-machine work. Phase 32+ candidate.
- **swift-content-encoding v0.7 streaming-decode wiring** — blocked on gzip+zlib+brotli all having streaming decode. Phase 33+.
- **swift-deflate v0.6 true memory-streaming inflate** — v0.5 just shipped; demand-driven.
- **swift-brotli/deflate v0.6 window carry** — ratio improvement; not blocking.
- **swift-oauth2-client v0.4** — v0.3 just shipped; no settle time.
- **B3 + Jaeger, idna v0.3** — no demand.
- **RS-family JWT** — STILL BLOCKED on swift-crypto `_RSA` SPI.
- **Package rename** — premature.

## Proposal

### Anchor: swift-gzip v0.5 + swift-zlib v0.5 streaming inflate (two-tranche phase)

Each tranche bumps an existing codec package's minor version and adds a `Streaming.Decoder` struct mirroring `Deflate.Streaming.Decoder`'s shape.

#### Tranche 31A: swift-gzip v0.5

Add `Gzip.Streaming.Decoder` namespace alongside v0.4's `Gzip.decode(_:)` and `Gzip.Streaming.Encoder`. Decoder buffers compressed input + decodes header + verifies CRC32 + ISIZE trailer at finish.

| Addition | Description |
|---|---|
| `Gzip.Streaming.Decoder` struct | Sendable value-type streaming decoder. Wraps Deflate.Streaming.Decoder. |
| `init()` | No parameters. |
| `update(_ chunk: Bytes)` | Buffers compressed input. Empty/finished no-op. |
| `finish() throws(GzipError) -> Bytes` | Parses gzip header/trailer, runs inner deflate decode on body bytes, verifies CRC32 + ISIZE. Returns decompressed output. Throws on header/trailer/checksum errors. |
| Possible new error case | `GzipError.decoderFinished` (mirror brotli/deflate pattern) — TBD by brainstorm. |

**Reality-check items:**
- Survey existing v0.1 `Decoder.swift` (gzip header/trailer parsing logic). v0.5 Streaming.Decoder reuses the same parsing in finish-time.
- Settle parsing strategy: buffer all input + parse header/body/trailer at finish (matches deflate v0.5 buffering wrap; simplest) vs incremental header parse + feed remainder to inner deflate decoder.
- Settle multi-member gzip streams: v0.1 decoder accepts them; v0.5 streaming should too.
- Reuse GzipError cases for v0.1 errors (badMagic, unsupportedCompressionMethod, crc32Mismatch, isizeMismatch, malformedDeflate, etc.).

#### Tranche 31B: swift-zlib v0.5

Add `Zlib.Streaming.Decoder` namespace alongside v0.4's `Zlib.decode(_:)` and `Zlib.Streaming.Encoder`. Decoder buffers compressed input + decodes 2-byte header + verifies 4-byte ADLER32 trailer at finish.

| Addition | Description |
|---|---|
| `Zlib.Streaming.Decoder` struct | Sendable value-type streaming decoder. Wraps Deflate.Streaming.Decoder. |
| `init()` | No parameters. |
| `update(_ chunk: Bytes)` | Buffers compressed input. Empty/finished no-op. |
| `finish() throws(ZlibError) -> Bytes` | Parses zlib header, runs inner deflate decode on body bytes, verifies ADLER32 trailer. Returns decompressed output. |
| Possible new error case | `ZlibError.decoderFinished` — TBD by brainstorm. |

**Reality-check items:**
- Survey existing v0.1 `Decoder.swift` (zlib header parsing + ADLER32 verification).
- Settle parsing strategy (likely buffering wrap; matches Tranche 31A).
- Reuse ZlibError cases.

### Buffering-wrap pattern inheritance

Both tranches inherit Phase 30's Option B (buffering wrap) semantics:
- `update(_:)` buffers input internally.
- `finish()` runs the full v0.1 one-shot decode at finalization time.
- True memory-streaming inflate is deferred to v0.6+ (depends on deflate v0.6 state-machine refactor).
- CHANGELOG documents this honestly per the honest-scope-under-limitation pattern (Phase 25 → Phase 28 → Phase 30 instances).

### Test surface (~12-15 tests per tranche)

| Test category | Scope |
|---|---|
| Round-trip via v0.2 one-shot encoder | Encode "hello"/"hello world"/pangram/70 KiB via Gzip.encode (or Zlib.encode) → stream-decode → bytes match | 4-5 tests |
| Multi-chunk input | Split a one-shot-encoded output into 2/3/many chunks → stream-decode → bytes match | 2-3 tests |
| Header/trailer correctness | Valid 10-byte gzip header (with FNAME flag); valid 2-byte zlib header (CMF/FLG check); CRC32 mismatch error; ISIZE mismatch error; ADLER32 mismatch error | 3-4 tests |
| Multi-member gzip streams | Concatenated gzip members decode to concatenated output (gzip only) | 1-2 tests |
| Error cases | Truncated input throws .truncated; double-finish throws .decoderFinished; update-after-finish no-op | 3 tests |
| Edge cases | Single-byte payload; empty payload; FNAME-flagged gzip stream | 2-3 tests |

Estimated 12-15 new tests per tranche. Total Phase 31 tests added: ~24-30.

### Acceptance criteria (per tranche)

- v0.5.0 ships for the target package with `Streaming.Decoder` added.
- v0.1-v0.4 APIs unchanged.
- Stream-decoded output equals one-shot `Gzip.decode(_:)` / `Zlib.decode(_:)` output for all test cases.
- swift-deflate dep bumped 0.4 → 0.5 (for `Deflate.Streaming.Decoder`).
- CI green on macOS + Linux, first try.
- DocC includes `### Streaming decompress (v0.5+)` topic group.
- CHANGELOG + README updated with streaming-decode example + honest-scope note.

### Out of scope (deferred to v0.6+)

- True memory-streaming inflate (inherits from deflate v0.6 state-machine refactor).
- `reset()` for decoder reuse.
- Async/AsyncSequence wrapper.

### Migration (v0.4 → v0.5 per package)

**Additive only — non-breaking.** All v0.1-v0.4 APIs unchanged.

### Risk

**Low. Wrapper-pattern over Phase 30's deflate v0.5 streaming-decode.**

| Risk | Mitigation |
|---|---|
| Multi-member gzip streams need careful buffering (each member has its own header/trailer) | Reuse v0.1 Decoder.swift's multi-member loop in finish-time. Tested. |
| Adopters expect true memory-streaming and get buffering wrap | Document honestly in CHANGELOG (matches Phase 30 pattern). |
| Multi-coding HTTP body decoding will need both gzip and zlib streaming decoders ready | Phase 31 ships both in one phase — content-encoding v0.7 wiring (Phase 33+) is unblocked. |
| Test coverage for streaming-decode equivalent to one-shot | Round-trip via v0.2 one-shot encoder → stream-decode → byte-equal check is the gate. |

## Alternatives considered

See [Gate 30 retrospective](../docs/gates/2026-05-17-gate-30-retrospective.md) § Item 5 for the full candidate matrix.

## Dependencies

- swift-gzip v0.5 bumps swift-deflate dep 0.4 → 0.5 (for `Deflate.Streaming.Decoder`).
- swift-zlib v0.5 bumps swift-deflate dep 0.4 → 0.5.
- No new dependencies.

## References

- [RFC 1952](https://www.rfc-editor.org/rfc/rfc1952) — gzip file format.
- [RFC 1950](https://www.rfc-editor.org/rfc/rfc1950) — zlib compressed data format.
- [Gate 30 retrospective](../docs/gates/2026-05-17-gate-30-retrospective.md) — anchor decision rationale.
- [RFC-0035](0035-phase-30-anchor-swift-deflate-v0.5-streaming-inflate.md) — Phase 30 anchor (deflate v0.5 streaming inflate; the canonical Streaming.Decoder shape that gzip + zlib mirror).
- [RFC-0029](0029-phase-24-anchor-gzip-zlib-v0.3-streaming-encoders.md) — Phase 24 anchor (gzip + zlib v0.3 streaming encoders); the symmetric encode-side template.

## Decision

**Accepted 2026-05-17.** Phase 31 anchored on **swift-gzip v0.5 + swift-zlib v0.5 streaming inflate (2-tranche phase)**. Brainstorm + plan + execute via the bare-swift inline-execution pattern per tranche.

Per-tranche calendar estimate **deferred to brainstorm reality-check** per Gate 22's canonical pattern (~30-60 min each; ~1-2 hours total).

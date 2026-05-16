# RFC-0032 — Phase 27 anchor: codec-tier v0.4 drain() API sweep (4-tranche phase)

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-16 |
| Resolution | Accepted 2026-05-16 — Phase 27 plans may begin authoring. Four existing-package tranches: 27A brotli v0.4, 27B deflate v0.4, 27C gzip v0.4, 27D zlib v0.4. |

## Summary

Anchor Phase 27 on a **coordinated 4-tranche codec-tier v0.4 sweep** adding `drain() -> Bytes` to the `Streaming.Encoder` types in swift-brotli, swift-deflate, swift-gzip, and swift-zlib. `drain()` returns the accumulated bytes from the internal `BitWriter` buffer **without** terminating the stream or byte-aligning, allowing callers to pipe output incrementally during `update(_:)` cycles instead of buffering everything until `finish()`. Unblocks multi-coding streaming in swift-content-encoding v0.6 (Phase 28+).

Per-tranche scope is small (~30-45 min, wrapper-pattern calibration). Total Phase 27 ~2-3 hours wall-clock.

## Problem

[Gate 26 retrospective](../docs/gates/2026-05-16-gate-26-retrospective.md) item 5 requires "Phase 27 anchor decision recorded as an RFC" before Phase 27 plans begin. The retrospective surveyed nine candidate waves; this RFC formalizes the choice.

**Concrete gap:** swift-content-encoding v0.5 (Phase 25A, shipped 2026-05-16) explicitly throws `ContentEncodingError.multipleCodingsNotStreamable` at init for multi-coding headers like `"gzip, br"`. The reason: current `Streaming.Encoder` types in all 4 codec packages emit output bytes only at `finish()`, not incrementally during `update(_:)`. Composing them in a chain (e.g., gzip-encode → br-encode) requires buffering each encoder's full output before feeding it to the next — which defeats streaming. Gate 25 documented this as a Phase 26+ candidate via a codec-tier `drain()` API addition; Gate 26 reaffirms.

**Why now (Phase 27):**
1. Closes Phase 25's documented multi-coding deferral (2 deferrals across Gates 25 + 26).
2. Completes the streaming HTTP body story (encode side): Phase 22-25 wired single-coding streaming; Phase 27 + Phase 28 close the multi-coding loop.
3. Small per-tranche scope (wrapper-pattern calibration: 30-60 min each).
4. Canonical streaming-encoder shape applies (4/4 instances). drain() is an additive method that fits cleanly.
5. Audience continuity from Phases 22-25 codec sweep.

**Why not the alternatives:**
- **Streaming decoders sweep** — 12-20hr coordinated sweep; major Decoder.swift refactor; defer until adopter demand clearer.
- **swift-oauth2-client v0.3** — v0.2 just shipped same-day (zero settle time).
- **swift-brotli v0.4 + swift-deflate v0.4 window carry** — could integrate into Phase 27 as optional fifth+sixth tranches; defer to keep scope tight.
- **swift-distributed-tracing-bridge v0.3** (B3 + Jaeger) — no concrete demand.
- **swift-idna v0.3** — rarely-hit Unicode rules; correct deferral, not procedural drift.
- **RS-family JWT** — **STILL BLOCKED** on swift-crypto's `_RSA` SPI.
- **Package rename swift-jwt-verify → swift-jwt** — premature.

## Proposal

### Anchor: codec-tier v0.4 drain() API sweep (4-tranche phase)

Each tranche bumps an existing codec package's minor version and adds a single new method:

```swift
public mutating func drain() -> Bytes
```

to the `Streaming.Encoder` type.

#### Drain semantics (locked at RFC time)

- **Returns** all bytes currently accumulated in the internal `BitWriter` buffer (the byte-aligned portion).
- **Resets** the internal byte buffer to empty.
- **Preserves** the partial-byte buffer (the up-to-7 bits not yet on a byte boundary). The encoder state continues as if `drain()` was never called — `update(_:)` and `finish()` produce the same final bytes as a non-draining stream would have produced.
- **Does NOT byte-align.** Aligning would corrupt the stream by inserting zero bits mid-block.
- **Does NOT emit a terminator block.** That's `finish()`'s job.
- **Silent no-op on finished encoder** — returns empty `Bytes`. (Matches `update(_:)` semantics.)
- **Multiple drains are valid.** Bytes flow continuously until `finish()` returns the tail + terminator.
- **Concatenation property:** for any sequence of `update(_:)` + `drain()` + `update(_:)` + `drain()` + ... + `finish()` calls, the concatenated output bytes equal the bytes a non-draining stream would have produced.

#### Tranches

| Tranche | Package | Scope | Estimated calendar (reality-check-locked) |
|---|---|---|---|
| 27A | swift-brotli v0.4 | `Brotli.Streaming.Encoder.drain() -> Bytes` + `BitWriter.drain() -> Bytes` (internal) + ~5 tests | ~30-45 min |
| 27B | swift-deflate v0.4 | `Deflate.Streaming.Encoder.drain() -> Bytes` + `BitWriter.drain() -> Bytes` (internal) + ~5 tests | ~30-45 min |
| 27C | swift-gzip v0.4 | `Gzip.Streaming.Encoder.drain() -> Bytes` (calls inner `Deflate.Streaming.Encoder.drain()`; CRC32 + ISIZE state unchanged) + ~5 tests | ~30-45 min |
| 27D | swift-zlib v0.4 | `Zlib.Streaming.Encoder.drain() -> Bytes` (calls inner `Deflate.Streaming.Encoder.drain()`; ADLER32 state unchanged) + ~5 tests | ~30-45 min |

**Per-tranche brainstorm reality-check:**
- Confirm `BitWriter` internals expose accumulated bytes cleanly (partial-byte buffer survives drain).
- Confirm `gzip` + `zlib` drain calls inner `Deflate.Streaming.Encoder.drain()` directly; checksum state continues across drains (the inner drain doesn't reset checksums, since checksums are computed over uncompressed input which already flows through).
- Confirm tests: multiple drains preserve total byte equality to single-finish; drained bytes + finish-bytes round-trip via the v0.1 decoder.

**Per-tranche test plan (~5 tests each):**
1. `drain()` on a fresh encoder (no updates) returns empty Bytes.
2. `update(chunk) + drain()` returns at least some bytes (the metablock from the chunk).
3. `update(chunk) + drain() + finish()` concatenated equals `update(chunk) + finish()` byte-for-byte.
4. Multiple `update + drain` cycles followed by `finish()` produce a valid stream that round-trips via the decoder.
5. `drain()` on finished encoder returns empty Bytes (silent no-op).

### Acceptance criteria (per tranche)

- v0.4.0 ships for the target package with `Streaming.Encoder.drain() -> Bytes` added.
- v0.3 APIs unchanged (drain is additive method).
- v0.3 byte-for-byte preserved when `drain()` is never called (regression-tested).
- Stream output from `update + drain + ... + finish` decodes correctly via the v0.1 decoder.
- CI green on macOS + Linux, first try.
- DocC includes `drain()` in the topic group.
- CHANGELOG + README updated with drain example.

### Out of scope (deferred to Phase 28+)

- **swift-content-encoding v0.6 multi-coding streaming wiring** — depends on Phase 27 shipping all 4 drain APIs. Phase 28 candidate.
- **swift-brotli v0.4 + swift-deflate v0.4 window carry across chunks** — could pair as Phase 28+ if ratio improvement becomes priority.
- **Streaming decoders sweep** — Phase 29+ candidate.
- **`reset()` for encoder reuse** — Phase 30+.
- **Explicit per-chunk flush API** (force byte-alignment mid-stream) — Phase 30+.

### Migration (v0.3 → v0.4 per package)

**Additive only — non-breaking.** All v0.3 APIs unchanged. `drain()` is an additive method.

### Risk

**Low. Pure additive method on canonical streaming-encoder shape.**

| Risk | Mitigation |
|---|---|
| `BitWriter` internal state may not cleanly split bytes-vs-partial-byte | Brainstorm 27A confirms; spec includes locked semantics. |
| gzip + zlib outer-encoder drain semantics: inner drain returns DEFLATE bytes; checksums must continue | Documented in spec — checksum digests update on `update(_:)`, not on inner drain. Inner drain returns intermediate DEFLATE bytes; outer drain returns those bytes verbatim (gzip header is emitted at init or first drain; trailers are emitted at `finish()` only). |
| `drain()` semantics surprise — caller expects byte-aligned chunks for ad-hoc framing | Documented: drain does NOT byte-align. Stream-format-aware callers expecting byte alignment must use `finish()` + new-encoder cycle (not v0.4 scope). |
| Multi-coding wiring (Phase 28) hits edge cases | Phase 28 brainstorm reality-checks before scope-locking; current Phase 27 only ships drain API. |

## Alternatives considered

See [Gate 26 retrospective](../docs/gates/2026-05-16-gate-26-retrospective.md) § Item 5 for the full candidate matrix.

## Dependencies

- No new direct dependencies on any tranche.
- Phase 28 will bump swift-content-encoding's gzip/zlib/brotli deps to 0.4+ to access the new drain API.

## References

- [RFC 1951](https://www.rfc-editor.org/rfc/rfc1951) (DEFLATE), [RFC 1952](https://www.rfc-editor.org/rfc/rfc1952) (gzip), [RFC 1950](https://www.rfc-editor.org/rfc/rfc1950) (zlib), [RFC 7932](https://www.rfc-editor.org/rfc/rfc7932) (Brotli).
- [Gate 26 retrospective](../docs/gates/2026-05-16-gate-26-retrospective.md) — anchor decision rationale.
- [RFC-0027](0027-phase-22-anchor-swift-brotli-v0.3-streaming-encoder.md), [RFC-0028](0028-phase-23-anchor-swift-deflate-v0.3-streaming-encoder.md), [RFC-0029](0029-phase-24-anchor-gzip-zlib-v0.3-streaming-encoders.md) — codec-tier v0.3 streaming RFCs that Phase 27 extends with drain().
- [RFC-0030](0030-phase-25-anchor-swift-content-encoding-v0.5-streaming.md) — Phase 25 anchor that explicitly defers multi-coding streaming pending drain API.

## Decision

**Accepted 2026-05-16.** Phase 27 anchored on **codec-tier v0.4 drain() API sweep (4-tranche phase)**. Brainstorm + plan + execute via the bare-swift inline-execution pattern per tranche.

Per-tranche calendar estimate **deferred to brainstorm reality-check** per Gate 22's canonical pattern (~30-45 min each expected; ~2-3 hours total).

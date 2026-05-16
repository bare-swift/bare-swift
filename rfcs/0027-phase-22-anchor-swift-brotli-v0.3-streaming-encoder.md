# RFC-0027 — Phase 22 anchor: swift-brotli v0.3 (streaming encoder)

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-16 |
| Resolution | Accepted 2026-05-16 — Phase 22 plans may begin authoring. Single existing package: swift-brotli v0.3. |

## Summary

Anchor Phase 22 on **swift-brotli v0.3** — additive non-breaking minor version bump adding a streaming-encoder API (`Brotli.Streaming.Encoder` with `init(quality:)` / `update(_:)` / `finish()`) on top of v0.2's one-shot `Brotli.compress(_:quality:)`. Multi-metablock partitioning + chunk-boundary state carry-over. Closes the longest-standing open deferral in the codec tier (9 deferrals since Phase 12). Non-breaking — all v0.1 decode and v0.2 compress APIs unchanged.

Single-tranche; ~400-700 LOC; estimated ~3-5 hours wall-clock per Gate 21's streaming-state-machine calibration (longer than recent additive minor bumps because streaming introduces inherent state-machine complexity).

## Problem

[Gate 21 retrospective](../docs/gates/2026-05-16-gate-21-retrospective.md) item 5 requires "Phase 22 anchor decision recorded as an RFC" before Phase 22 plans begin. The retrospective surveyed seven candidate waves; this RFC formalizes the choice.

**Concrete gap:** swift-brotli v0.2 (Phase 12A, shipped 2026-05-13) ships a one-shot encoder: `Brotli.compress(_ bytes: Bytes, quality: Quality) throws(BrotliError) -> Bytes` that emits one Brotli metablock per call and caps input at 16 MiB. The v0.2 CHANGELOG lists "Streaming encoder API. v0.2 is one-shot only." as the **first explicit non-goal deferred to v0.3+**.

The deferral has been carried across **nine consecutive gates** (Phases 13, 14, 15, 16, 17, 18, 19, 20, 21). This crosses the procedural-correction threshold established in Gate 19 (JWT signing, 7 deferrals → Phase 19 breakthrough) and Gate 20 (OAuth, 8 deferrals → Phase 20 breakthrough). Per the now-canonical pattern: **after 7-9 rejections under a self-perpetuating "no concrete adopter demand" defer criterion, re-examine the criterion.** bare-swift is a proactive ecosystem build; the deferral has accumulated past the breakthrough threshold.

**Why now (Phase 22):**
1. The procedural-correction pattern applies at 9 deferrals (matches/exceeds Phase 19's 7 and Phase 20's 8).
2. Codec tier has zero streaming codec — deflate/gzip/zlib/brotli all one-shot. Brotli leads the streaming wave because it has the most primitives in place (v0.2 encoder is already decomposed into BitWriter, MatchFinder, EncoderCommand, HuffmanBuilder, PrefixCodeEmitter, EncoderMetaBlock).
3. swift-content-encoding v0.4 (HTTP `Content-Encoding: br` request-body streaming) is a natural follow-on; this RFC unlocks it.
4. v0.2 caps input at 16 MiB. Streaming naturally lifts this cap (each chunk is bounded, total stream unbounded), which is a quality-of-life win for any HTTP/WebSocket adopter.

**Why not the alternatives:**
- **swift-oauth2-client v0.2** (auth URL builder + PKCE) — v0.1 has only ~12 hours real-world settle time. Phase 23+ candidate.
- **swift-distributed-tracing-bridge v0.3** (B3 + Jaeger) — no concrete demand; W3C just landed in Phase 18.
- **swift-idna v0.3** (Bidi + ContextJ + ContextO) — rarely-hit rules; no demand.
- **RS-family JWT** (RS256/RS384/RS512/PS256) — **blocked** on swift-crypto's `_RSA` SPI stabilization; cannot ship safely until upstream API stabilizes.
- **Package rename swift-jwt-verify → swift-jwt** — premature without paired-feature change. Phase 24+ candidate.

## Proposal

### Anchor: swift-brotli v0.3 (single existing package, additive minor bump)

**swift-brotli v0.3** — add a streaming `Brotli.Streaming.Encoder` namespace alongside v0.2's `Brotli.compress(_:quality:)`. v0.2's one-shot encoder remains unchanged and is the recommended path for ≤16 MiB inputs; streaming is the new path for unbounded / chunked / HTTP-streaming workloads.

| Addition | Description |
|---|---|
| `Brotli.Streaming.Encoder` struct | Sendable + value-type-ish (state-bearing; documented usage rules) streaming encoder. |
| `init(quality: Quality = .default)` | Same `Quality` enum as v0.2. |
| `mutating func update(_ chunk: Bytes)` | Feed a chunk; encoder may emit zero or more metablocks internally. |
| `mutating func finish() throws(BrotliError) -> Bytes` | Emit pending bytes + the terminator (ISLAST=1) metablock. Returns the full compressed output. |
| Internal: multi-metablock orchestration | One metablock per `update(_:)` chunk OR window-bounded chunking — see § 4. |
| Internal: distance ring-buffer carry-over | Brotli's last-4-distance ring buffer (§ 4) must carry across metablock boundaries to preserve compression ratio when the same distances recur. |
| Internal: window carry across chunks | Optional but recommended — see § 4 for Shape A vs Shape B. |
| Possible new error cases | `BrotliError.encoderFinished` (calling `update` after `finish`) or similar — see § 5. |

### Key bytes / API shape (subject to brainstorm refinement)

```swift
// v0.3 streaming usage (proposed):
import Brotli
import Bytes

var encoder = Brotli.Streaming.Encoder(quality: .default)
encoder.update(chunk1)
encoder.update(chunk2)
let compressed = try encoder.finish()

// Decode (unchanged from v0.1):
let plain = try Brotli.decode(compressed)
```

The decoder is unchanged. The streaming encoder's output is **valid Brotli** that decodes via the same `Brotli.decode(_:)` v0.1 API.

### Decomposition (subject to brainstorm)

Two shapes were considered:

**Shape A (recommended): single tranche, full streaming with window carry.**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 22A | swift-brotli v0.3 | `Brotli.Streaming.Encoder` (init/update/finish) + multi-metablock partitioning + chunk-boundary distance-ring-buffer carry-over + window carry across metablocks for compression-ratio preservation + ~25-30 tests | ~3-5 hours |

**Shape B (alternate): two tranches if window-carry semantics introduce surprises.**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 22A | swift-brotli v0.3 | Streaming API + multi-metablock partitioning + distance-ring-buffer carry — but each metablock independent (no LZ77 match search across chunks) | ~2-3 hours |
| 22B | swift-brotli v0.4 | Window carry across metablocks (full LZ77 across stream boundary) for higher compression ratio | ~2-3 hours |

Decision deferred to brainstorm phase. Per the brainstorm-reality-check pattern, expect Shape B only if window-carry introduces surprises in `Sources/Brotli/MatchFinder.swift` or `Sources/Brotli/Encoder.swift` — the v0.2 MatchFinder may not have hooks for window carry across "session" boundaries.

### Test surface

| Test category | Scope |
|---|---|
| Round-trip: stream encode → flatten → v0.1 decode = original input | 5-10 tests across chunk shapes (single chunk = matches v0.2 one-shot byte-for-byte, two equal chunks, many 1-byte chunks, alternating large/small chunks, empty stream, finish-without-update) |
| Quality-level coverage | 3-5 tests across `.fastest` / `.default` / `.smallest` |
| Boundary cases | Empty stream → valid empty Brotli, single byte stream, exactly-16-MiB single-shot equivalence, multi-chunk past 16-MiB total (cap-lifting check) |
| Error cases | Calling `update` after `finish` throws; double `finish` (if supported, either no-op or throws — to be decided in brainstorm) |
| Distance carry-over | Encode two chunks where chunk2 has a long match to chunk1's tail; verify chunk2's compressed output references the carried distance (compression-ratio check, not byte-equality) |
| Reference-encoder parity (not asserted) | Same as v0.2: output is valid Brotli that decodes correctly, but ratio not asserted against reference encoder |

Estimated 25-30 new tests; total package tests post-22A: ~125-130 (up from ~103).

### Acceptance criteria

- v0.3.0 ships with `Brotli.Streaming.Encoder` (init/update/finish).
- v0.1 + v0.2 APIs unchanged. v0.2's one-shot `Brotli.compress(_:quality:)` continues to byte-for-byte produce v0.2 output (regression test).
- Stream-encoded output decodes correctly via v0.1 `Brotli.decode(_:)` for all test cases.
- CI green on macOS + Linux, all 7 jobs, first try (continuing the 6-phase first-try streak).
- DocC includes a `### Streaming (v0.3+)` topic group with usage examples.
- CHANGELOG + README updated with streaming example.

### Out of scope (deferred to v0.4+)

- **Streaming decode.** v0.1 decoder is already one-shot; streaming decode is the natural v0.4 deferral. (Compression-asymmetric pattern: streaming encode covers HTTP request bodies; streaming decode covers HTTP response bodies — caller's choice which they need.)
- **Static-dictionary search in streaming encoder.** Same v0.2 deferral; the static dictionary stays decode-only.
- **Multi-thread streaming.** Brotli reference allows parallelization at metablock boundaries; out of scope.
- **`reset()` method for stream reuse.** v0.3 ships one-shot-per-instance (after `finish`, encoder is consumed). `reset()` is a v0.4 candidate.
- **Explicit flush API.** v0.3 ships finalize-only. Per-chunk flush (forced emission for latency-sensitive streaming) is a v0.4 candidate.
- **RS-family JWT** in swift-jwt-verify (Phase 23+ when swift-crypto stabilizes `_RSA`).
- **swift-content-encoding v0.4 streaming wiring** (natural Phase 23 follow-on).

### Migration (v0.2 → v0.3)

**Additive only — non-breaking.** All v0.2 APIs unchanged:
- `Brotli.compress(_:quality:)` continues to work for ≤16 MiB inputs.
- `Brotli.decode(_:)` unchanged from v0.1.
- `BrotliError` adds 1-2 new cases (additive; existing cases unchanged).
- `Brotli.Quality` unchanged.

Adopters opt into streaming by switching from `Brotli.compress(_:)` to `Brotli.Streaming.Encoder(quality:)` + chunked `update` + `finish`.

### Risk

**Medium — higher than recent additive phases (which have been Low).**

| Risk | Mitigation |
|---|---|
| Brotli metablock semantics around state carry (distance ring, window) have subtle interactions | Brainstorm reads RFC 7932 § 4 + § 9.2 carefully; verifies v0.2's `EncoderMetaBlock` already produces compliant metablocks |
| `MatchFinder` may not have a clean hook for window carry across chunks | Brainstorm surveys v0.2's `MatchFinder.swift` for chunk-boundary handling. If clean → Shape A. If complex → Shape B. |
| Output ratio regression vs v0.2 one-shot when stream is fed as one chunk | Test: feed v0.3 streaming encoder a single chunk, assert output ≈ v0.2 one-shot bytes (byte-equal for matching internal partition decisions; otherwise size-comparable within ±5% slack) |
| Chunk-boundary `update` mutating semantics across Sendable boundary | `Brotli.Streaming.Encoder` is not Sendable (it's a mutating state machine). Documented and enforced — likely makes it a non-Sendable value type. |
| `finish` could throw on internal state error | Existing `BrotliError` covers; possibly add `encoderFinished` for "double finish" or "update after finish". |

**Risk profile is bounded:** the v0.2 encoder is already decomposed into reusable primitives (BitWriter, Encoder, MatchFinder, EncoderCommand, HuffmanBuilder, PrefixCodeEmitter, EncoderMetaBlock). v0.3 adds the orchestration layer + chunk-boundary state. No new core algorithms.

**Bounded calendar estimate (~3-5 hours):** larger than the recent additive-minor-bump phases (~1 hour) because streaming state machines have inherent complexity. Per the Gate 20-21 calibration: hours for bounded scope, but streaming is not a one-step expansion — it's a new architectural layer over existing primitives.

## Alternatives considered

See [Gate 21 retrospective](../docs/gates/2026-05-16-gate-21-retrospective.md) § Item 5 for the full candidate matrix. Summary:

- **swift-oauth2-client v0.2** — v0.1 too fresh (~12 hours). Phase 23+.
- **swift-distributed-tracing-bridge v0.3** — no concrete demand. Phase 23+.
- **swift-idna v0.3** — rarely-hit rules; no demand. Phase 23+.
- **RS-family JWT** — blocked on swift-crypto `_RSA` SPI. Cannot ship safely.
- **Package rename swift-jwt-verify → swift-jwt** — premature without paired-feature change. Phase 24+ candidate.
- **Crypto-adjacent waves** — 17th consecutive rejection.

## Dependencies

- No new direct dependencies. swift-brotli v0.2 already depends on swift-bytes.
- No new transitive dependencies.

## References

- [RFC 7932](https://www.rfc-editor.org/rfc/rfc7932) — Brotli compressed data format. § 4 (distance codes / ring buffer), § 9 (stream structure / metablock structure).
- [Gate 21 retrospective](../docs/gates/2026-05-16-gate-21-retrospective.md) — anchor decision rationale + Phase 22 candidate survey.
- [RFC-0001](0001-conventions-and-quality-bars.md) — bare-swift conventions (Sendable, Foundation-free public, typed errors, etc.).
- [RFC-0017](0017-phase-12-anchor-swift-brotli-v0.2-encoder.md) — Phase 12 anchor RFC for v0.2 encoder; defines `Brotli.compress(_:quality:)` v0.2 surface that v0.3 augments.
- swift-brotli v0.2 CHANGELOG — explicit v0.3+ non-goals list (§ "v0.2 encoder scope (explicit non-goals — deferred to v0.3+)").
- Phase 19 (JWT signing 7-rejection breakthrough) + Phase 20 (OAuth 8-rejection breakthrough) — establishing the 7-9 deferral procedural-correction threshold that Phase 22 (9-rejection brotli streaming) closes.

## Decision

**Accepted 2026-05-16.** Phase 22 anchored on **swift-brotli v0.3 streaming encoder**. Brainstorm + plan + execute via the bare-swift inline-execution pattern (spec → plan → coalesced inline implementation → tag → umbrella bump → memory closure). Shape A (single tranche, full streaming with window carry) is the working assumption; Shape B remains a fallback if brainstorm uncovers `MatchFinder` boundary surprises.

# RFC-0028 — Phase 23 anchor: swift-deflate v0.3 (streaming encoder)

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-16 |
| Resolution | Accepted 2026-05-16 — Phase 23 plans may begin authoring. Single existing package: swift-deflate v0.3. |

## Summary

Anchor Phase 23 on **swift-deflate v0.3** — additive non-breaking minor version bump adding a streaming-encoder API (`Deflate.Streaming.Encoder` with `init(level:)` / `update(_:)` / `finish()`) on top of v0.2's one-shot `Deflate.encode(_:level:)`. Mirrors Phase 22's `Brotli.Streaming.Encoder` shape (Sendable struct + State enum + init/update/finish + throwing-finish-on-double-call + silent-no-op-on-update-after-finish). Closes documented v0.3 deferral from Phase 9. Continues codec-tier streaming sweep started in Phase 22; unblocks swift-gzip / swift-zlib / swift-content-encoding streaming in subsequent phases.

Single-tranche; estimated calendar **determined by brainstorm reality-check** (RFC-0027 over-estimated 3-5x because streaming-complexity was assumed; Phase 23 RFC defers the estimate to the spec phase per the canonical reality-check-before-RFC-estimate pattern codified in Gate 22).

## Problem

[Gate 22 retrospective](../docs/gates/2026-05-16-gate-22-retrospective.md) item 5 requires "Phase 23 anchor decision recorded as an RFC" before Phase 23 plans begin. The retrospective surveyed nine candidate waves; this RFC formalizes the choice.

**Concrete gap:** swift-deflate v0.2 (Phase 9A, shipped 2026-05-11) ships a one-shot encoder: `Deflate.encode(_:level:) -> Bytes` that takes a complete `Bytes` input and produces a complete RFC 1951 DEFLATE bit stream. The v0.2 CHANGELOG explicitly says "single-shot encoder (streaming ships in v0.3 per RFC-0014)." Phase 23 closes this documented v0.3 deferral.

**Why now (Phase 23):**
1. Phase 22 shipped swift-brotli v0.3 streaming. Codec-tier streaming sweep continues: brotli → deflate → gzip + zlib → content-encoding.
2. DEFLATE is the most-used HTTP codec. gzip wraps DEFLATE; zlib wraps DEFLATE; raw DEFLATE is the base. Streaming DEFLATE unblocks three downstream packages.
3. Streaming-encoder shape is now canonical from Phase 22. Phase 23 reuses the same Sendable-struct + State-enum + init/update/finish template — no new architectural decisions.
4. Brainstorm reality-check pattern is canonical (Gate 22 codification). Phase 23 brainstorm surveys swift-deflate v0.2's `Deflate.Encoder` source for stateful interactions before locking scope + estimate.

**Why not the alternatives:**
- **swift-gzip v0.3 streaming + swift-zlib v0.3 streaming** — both wrap swift-deflate, which doesn't have streaming yet. Phase 23 must ship deflate streaming first; gzip + zlib follow in Phase 24+.
- **swift-content-encoding v0.5 streaming wiring** — blocked on deflate/gzip/zlib all having streaming. Phase 25+ candidate.
- **swift-brotli v0.4 window carry** — ratio improvement only; lower urgency than unblocking the deflate-family codec sweep.
- **swift-oauth2-client v0.2** (auth URL builder + PKCE) — settle time approaching (~30 hours); strong Phase 24+ candidate, but codec-tier sweep continuity wins this gate.
- **swift-distributed-tracing-bridge v0.3** (B3 + Jaeger) — no concrete demand.
- **swift-idna v0.3** — rarely-hit rules; no demand.
- **RS-family JWT** (RS256/RS384/RS512/PS256) — **STILL BLOCKED** on swift-crypto's `_RSA` SPI stabilization. Re-check each gate.
- **Package rename swift-jwt-verify → swift-jwt** — premature without paired-feature change.

## Proposal

### Anchor: swift-deflate v0.3 (single existing package, additive minor bump)

**swift-deflate v0.3** — add a streaming `Deflate.Streaming.Encoder` namespace alongside v0.2's `Deflate.encode(_:level:)`. v0.2's one-shot encoder remains unchanged and is the recommended path for bounded inputs; streaming is the new path for unbounded / chunked / HTTP-streaming workloads.

| Addition | Description |
|---|---|
| `Deflate.Streaming` namespace enum | Public Sendable namespace. |
| `Deflate.Streaming.Encoder` struct | Sendable value-type streaming encoder. |
| `init(level: Deflate.Encoder.Level = .default)` | Same `Level` typealias as v0.2. Likely non-throwing (Level enum is closed; no runtime invalidation possible — pending brainstorm). |
| `mutating func update(_ chunk: Bytes)` | Feed a chunk; encoder emits one DEFLATE block per chunk (or N blocks if state-carry decision requires). |
| `mutating func finish() throws(DeflateError) -> Bytes` | Emit pending bytes + the final block (BFINAL=1). Returns the full DEFLATE bit stream. Throws on double-call. |
| Internal: multi-block orchestration | One DEFLATE block per `update(_:)` chunk OR window-bounded chunking — determined by reality-check. |
| Possible new error case | `DeflateError.encoderFinished` (mirror brotli's `encoderFinished`) — additive, pending brainstorm. |

### Key bytes / API shape (subject to brainstorm refinement)

```swift
// v0.3 streaming usage (proposed):
import Deflate
import Bytes

var encoder = Deflate.Streaming.Encoder(level: .default)
encoder.update(chunk1)
encoder.update(chunk2)
let compressed = try encoder.finish()

// Decode (unchanged from v0.1):
let plain = try Deflate.inflate(compressed)
```

The decoder is unchanged. The streaming encoder's output is **valid DEFLATE** that decodes via the same `Deflate.inflate(_:)` v0.1 API.

### Decomposition (subject to brainstorm reality-check)

Two shapes were considered:

**Shape A (recommended pending reality-check): single tranche, per-chunk independent blocks.**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 23A | swift-deflate v0.3 | `Deflate.Streaming.Encoder` (init/update/finish) + per-chunk DEFLATE-block emission (each block independent, no sliding-window carry) + ~12-18 tests | ~1-2 hours |

**Shape B (alternate): two tranches if sliding-window carry has non-trivial scope.**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 23A | swift-deflate v0.3 | Streaming API + per-chunk independent DEFLATE blocks (no LZ77 match search across chunks) | ~1-2 hours |
| 23B | swift-deflate v0.4 | Sliding-window carry across chunks (full LZ77 across stream boundary; reuse Huffman trees) for higher compression ratio | ~2-3 hours |

Decision deferred to brainstorm phase. Per the now-canonical reality-check pattern:
- If v0.2's `Deflate.Encoder` is **stateless across blocks** (like swift-brotli v0.2's EncoderMetaBlock — fresh Huffman trees per block, no sliding window carry), Shape A applies; estimate is ~1-2 hours.
- If v0.2 encoder **exploits sliding window** across blocks within a single `encode()` call, streaming requires either (a) accept ratio loss at chunk boundaries (Shape A; documented v0.4 deferral) or (b) carry window state across `update(_:)` calls (Shape B's 23B tranche).

RFC 1951's sliding window is up to 32 KiB. If v0.2 encoder emits one DEFLATE block per `encode()` call (likely; mirrors brotli v0.2's one-metablock-per-call pattern), Shape A is the natural fit.

### Test surface

| Test category | Scope |
|---|---|
| Round-trip: stream encode → flatten → v0.1 decode = original input | 5-8 tests across chunk shapes (single chunk = matches v0.2 one-shot byte-for-byte OR within byte-count slack, two chunks, many tiny chunks, empty stream, finish-without-update) |
| Level coverage | 3-4 tests across `.none` / `.fast` / `.default` / `.best` |
| Boundary cases | Empty stream → valid empty DEFLATE, single byte stream, large chunk (>1 MiB), mixed-size chunks |
| Error cases | Double-finish throws (likely `encoderFinished`); update-after-finish silent no-op |
| Equivalence | Single-update streaming output decodes to same bytes as `Deflate.encode(_:)` one-shot output |

Estimated 12-18 new tests; total package tests post-23A: depending on v0.2 baseline (likely ~50-70 → ~65-90).

### Acceptance criteria

- v0.3.0 ships with `Deflate.Streaming.Encoder` (init/update/finish).
- v0.1 + v0.2 APIs unchanged. v0.2's `Deflate.encode(_:level:)` continues to produce byte-for-byte identical output to v0.2 (regression test).
- Stream-encoded output decodes correctly via v0.1 `Deflate.inflate(_:)` for all test cases.
- CI green on macOS + Linux, all 7 jobs, first try (continuing the 7-phase first-try streak).
- DocC includes a `### Streaming (v0.3+)` topic group.
- CHANGELOG + README updated with streaming example.

### Out of scope (deferred to v0.4+)

- **Sliding-window carry across chunks.** If reality-check finds v0.2 doesn't already do this, defer to v0.4 for ratio improvement (same approach as swift-brotli v0.3 → v0.4).
- **Streaming inflate.** v0.1 decoder is one-shot; streaming decode is v0.4+ candidate.
- **`reset()` for encoder reuse.** Same as swift-brotli v0.3 — v0.4 candidate.
- **Explicit flush API.** v0.3 ships finalize-only.
- **Multi-thread streaming.**
- **Block-type optimization** (choose stored / fixed Huffman / dynamic Huffman per-chunk based on entropy). v0.2's choice unchanged in v0.3.

### Migration (v0.2 → v0.3)

**Additive only — non-breaking.** All v0.2 APIs unchanged:
- `Deflate.encode(_:level:)` continues to produce byte-for-byte identical output (regression-tested).
- `Deflate.inflate(_:)` unchanged from v0.1.
- `DeflateError` may add 1 new case (additive; existing cases unchanged).
- `Deflate.Encoder.Level` unchanged.

Adopters opt into streaming by switching to `Deflate.Streaming.Encoder`.

### Risk

**Medium — comparable to Phase 22's brotli streaming risk.**

| Risk | Mitigation |
|---|---|
| DEFLATE block-type semantics around state carry (sliding window, Huffman trees) have subtle interactions | Brainstorm reads RFC 1951 § 3.2 + surveys v0.2 `Deflate.Encoder` source for state usage |
| v0.2 encoder may exploit sliding window across blocks — streaming would need to either accept ratio loss OR carry window | Decided by reality-check; defer window carry to v0.4 if needed |
| `update(_:)` mutating semantics across Sendable boundary | `Deflate.Streaming.Encoder` non-Sendable... actually, brotli's solution: Sendable value-type with documented copy-divergence. Reuse pattern. |
| Output ratio regression vs v0.2 one-shot for single-chunk feed | Test: feed v0.3 streaming encoder a single chunk, assert decoded output equals input (decoded-equality, not byte-equality unless reality-check finds byte-equality is feasible) |
| Pre-existing `var` warnings in v0.2 surface during local `-warnings-as-errors` builds | Pattern reinforced in Phase 22; not a CI gate; track in separate quality-cleanup queue |

**Risk profile is bounded:** v0.2's encoder is already decomposed into primitives. v0.3 adds the orchestration layer + per-chunk block emission. No new core algorithms.

**Calendar estimate (~1-3 hours):** depends on reality-check finding. Per Gate 22's canonical pattern, the brainstorm spec phase locks the estimate based on v0.2's stateful-interaction survey, not the RFC.

## Alternatives considered

See [Gate 22 retrospective](../docs/gates/2026-05-16-gate-22-retrospective.md) § Item 5 for the full candidate matrix. Summary:

- **swift-gzip v0.3 streaming + swift-zlib v0.3 streaming** — blocked on deflate streaming. Phase 24+.
- **swift-content-encoding v0.5 streaming wiring** — blocked on full codec-tier sweep. Phase 25+.
- **swift-brotli v0.4 window carry** — ratio improvement; lower urgency.
- **swift-oauth2-client v0.2** — Phase 24+ candidate; settle time approaching.
- **swift-distributed-tracing-bridge v0.3 (B3 + Jaeger)** — no concrete demand.
- **swift-idna v0.3** — rarely-hit rules; no demand.
- **RS-family JWT** — blocked on swift-crypto `_RSA` SPI.
- **Package rename swift-jwt-verify → swift-jwt** — premature.
- **Crypto-adjacent waves** — 18th consecutive rejection.

## Dependencies

- No new direct dependencies. swift-deflate v0.2 already depends on swift-bytes.
- No new transitive dependencies.

## References

- [RFC 1951](https://www.rfc-editor.org/rfc/rfc1951) — DEFLATE compressed data format. § 3 (block structure), § 3.2 (compressed blocks).
- [Gate 22 retrospective](../docs/gates/2026-05-16-gate-22-retrospective.md) — anchor decision rationale + Phase 23 candidate survey + reality-check-before-RFC-estimate codification.
- [RFC-0001](0001-conventions-and-quality-bars.md) — bare-swift conventions.
- [RFC-0014](0014-phase-8-anchor-deflate-encoder.md) — Phase 8 anchor (deflate v0.2 encoder); defines `Deflate.encode(_:level:)` v0.2 surface that v0.3 augments.
- [RFC-0027](0027-phase-22-anchor-swift-brotli-v0.3-streaming-encoder.md) — Phase 22 (brotli streaming); template for Phase 23's streaming-encoder shape.
- swift-deflate v0.2 CHANGELOG — documents v0.3 streaming deferral.
- Phase 19 (JWT signing 7-rejection breakthrough) + Phase 20 (OAuth 8-rejection breakthrough) + Phase 22 (brotli streaming 9-rejection breakthrough) — establishing the 7-9 deferral procedural-correction threshold.

## Decision

**Accepted 2026-05-16.** Phase 23 anchored on **swift-deflate v0.3 streaming encoder**. Brainstorm + plan + execute via the bare-swift inline-execution pattern (spec → plan → coalesced inline implementation → tag → umbrella bump → memory closure). Shape A (single tranche, per-chunk independent blocks) is the working assumption; Shape B (window-carry as separate tranche) remains a fallback if brainstorm reality-check uncovers DEFLATE-specific cross-block state in v0.2's encoder.

Calendar estimate **deferred to brainstorm reality-check** per Gate 22's canonical pattern.

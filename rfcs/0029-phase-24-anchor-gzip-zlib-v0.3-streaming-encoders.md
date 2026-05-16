# RFC-0029 — Phase 24 anchor: swift-gzip v0.3 + swift-zlib v0.3 streaming encoders

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-16 |
| Resolution | Accepted 2026-05-16 — Phase 24 plans may begin authoring. Two existing packages, two tranches: swift-gzip v0.3 (24A) and swift-zlib v0.3 (24B). |

## Summary

Anchor Phase 24 on **swift-gzip v0.3 + swift-zlib v0.3** — two additive non-breaking minor version bumps adding `Streaming.Encoder` types to each package, mirroring Phase 22 brotli + Phase 23 deflate templates. Both encoders wrap `Deflate.Streaming.Encoder` (now available from Phase 23) + an incremental checksum (CRC32 for gzip; ADLER32 for zlib) + framing header/trailer emit. Completes the bare-swift codec-tier streaming sweep at the package level — by end of Phase 24, all four HTTP-codec packages (brotli, deflate, gzip, zlib) are stream-capable.

Two tranches; estimated calendar **determined by brainstorm reality-check** per the canonical Gate 22 pattern (~1-2 hours each; ~2-4 hours total).

## Problem

[Gate 23 retrospective](../docs/gates/2026-05-16-gate-23-retrospective.md) item 5 requires "Phase 24 anchor decision recorded as an RFC" before Phase 24 plans begin. The retrospective surveyed nine candidate waves; this RFC formalizes the choice.

**Concrete gaps:**
- **swift-gzip v0.2** (Phase 9B, shipped 2026-05-11) ships a one-shot encoder: `Gzip.encode(_:level:filename:modificationTime:) -> Bytes`. The v0.2 CHANGELOG explicitly says "single-shot encoder (streaming ships in v0.3)."
- **swift-zlib v0.2** (Phase 9C, shipped 2026-05-12) ships a one-shot encoder: `Zlib.encode(_:level:) -> Bytes`. Same v0.3 streaming deferral.

Both deferrals dated from Phase 9 and were unblocked by Phase 23's `Deflate.Streaming.Encoder`. With the streaming primitive now available, gzip and zlib streaming wrappers are tight (each ~100-150 LOC).

**Why now (Phase 24):**
1. Phase 22 + Phase 23 shipped brotli + deflate streaming. Codec-tier streaming sweep continues: brotli → deflate → **gzip + zlib** → content-encoding wiring.
2. swift-deflate v0.3 streaming is the dependency for both swift-gzip and swift-zlib streaming. **Available now** (Phase 23A shipped same-day).
3. Multi-tranche phase precedent: Phase 9 had 4 tranches (deflate v0.2, gzip v0.2, zlib v0.2, content-encoding v0.2). Phase 24's gzip + zlib pair maps cleanly to 2 tranches.
4. Same canonical template applies (Sendable struct + State enum + init/update/finish + double-finish-throws + update-after-finish-no-op). No new architectural decisions.
5. Audience continuity: HTTP-codec adopters from Phases 9, 10, 12, 22, 23.

**Why not the alternatives:**
- **swift-content-encoding v0.5 streaming wiring** — blocked on gzip + zlib streaming. Phase 25+.
- **swift-brotli v0.4 + swift-deflate v0.4 window carry** — v0.3 just shipped; not procedural drift. Phase 26+ candidate.
- **swift-oauth2-client v0.2** — settle time complete (~1 day exposure of v0.1); could ship, but codec-tier sweep completion has higher leverage.
- **swift-distributed-tracing-bridge v0.3** (B3 + Jaeger) — no concrete demand.
- **swift-idna v0.3** — rarely-hit rules; no demand.
- **RS-family JWT** (RS256/RS384/RS512/PS256) — **STILL BLOCKED** on swift-crypto's `_RSA` SPI stabilization. Re-check each gate.
- **Package rename swift-jwt-verify → swift-jwt** — premature.

## Proposal

### Anchor: swift-gzip v0.3 + swift-zlib v0.3 (two-tranche phase, two existing-package minor bumps)

#### Tranche 24A: swift-gzip v0.3

Add `Gzip.Streaming.Encoder` namespace alongside v0.2's `Gzip.encode(_:level:filename:modificationTime:)`. Existing one-shot Gzip.encode + Gzip.Encoder preserved verbatim.

| Addition | Description |
|---|---|
| `Gzip.Streaming` namespace enum | Public Sendable namespace. |
| `Gzip.Streaming.Encoder` struct | Sendable value-type streaming encoder. |
| `init(level:filename:modificationTime:)` | Mirror v0.2's one-shot init parameters. Emits 10-byte RFC 1952 header (with optional FNAME) into internal buffer. |
| `update(_ chunk: Bytes)` | Feed chunk to internal `Deflate.Streaming.Encoder`. Update incremental CRC32 over uncompressed bytes. Update ISIZE counter. |
| `finish() throws(GzipError) -> Bytes` | Finalize DEFLATE via inner `Deflate.Streaming.Encoder.finish()`. Append 8-byte trailer (CRC32 LE + ISIZE LE). Return full gzip stream. |
| Possible new error case | `GzipError.encoderFinished` for double-call. Pending brainstorm — may already be analog. |

**Reality-check items (brainstorm 24A):**
- Verify swift-crc's `CRC.compute(_:algorithm:)` API. If it has incremental state (`init` + `update` + `finalize`), use it. If not, either (a) add an incremental wrapper, or (b) accept a small refactor in swift-crc's public API.
- Verify how `Gzip.encode` calls `Deflate.encode` to confirm we can swap in `Deflate.Streaming.Encoder` cleanly.
- Confirm existing `Gzip.Encoder` struct + `Gzip.encode` function are preserved verbatim (existing-encoder-preserved-verbatim pattern from Gate 23).

#### Tranche 24B: swift-zlib v0.3

Add `Zlib.Streaming.Encoder` namespace alongside v0.2's `Zlib.encode(_:level:)`. Existing one-shot Zlib.encode + Zlib.Encoder preserved verbatim.

| Addition | Description |
|---|---|
| `Zlib.Streaming` namespace enum | Public Sendable namespace. |
| `Zlib.Streaming.Encoder` struct | Sendable value-type streaming encoder. |
| `init(level:)` | Mirror v0.2's one-shot init. Emits 2-byte RFC 1950 header (CMF + FLG with FCHECK computed). |
| `update(_ chunk: Bytes)` | Feed chunk to internal `Deflate.Streaming.Encoder`. Update incremental ADLER32 over uncompressed bytes. |
| `finish() throws(ZlibError) -> Bytes` | Finalize DEFLATE via inner `Deflate.Streaming.Encoder.finish()`. Append 4-byte ADLER32 trailer (BE per RFC 1950 § 2.2). Return full zlib stream. |
| Possible new error case | `ZlibError.encoderFinished` for double-call. Pending brainstorm. |

**Reality-check items (brainstorm 24B):**
- Verify `Adler32.swift` (20 LOC). ADLER32 is inherently rolling (s1 = (s1 + byte) mod 65521; s2 = (s2 + s1) mod 65521). v0.2's `Adler32.compute(_:) -> UInt32` is one-shot but trivially extensible to incremental (`init` + `update` + `finalize`).
- Confirm existing `Zlib.Encoder` struct + `Zlib.encode` function are preserved verbatim.

### Key bytes / API shape (subject to brainstorm refinement)

```swift
// 24A: swift-gzip v0.3
import Gzip
import Bytes

var encoder = Gzip.Streaming.Encoder(level: .default, filename: "data.txt", modificationTime: 1700000000)
encoder.update(chunk1)
encoder.update(chunk2)
let gzipped = try encoder.finish()
let plain = try Gzip.decode(gzipped)

// 24B: swift-zlib v0.3
import Zlib
import Bytes

var encoder = Zlib.Streaming.Encoder(level: .default)
encoder.update(chunk1)
encoder.update(chunk2)
let zlibbed = try encoder.finish()
let plain = try Zlib.decode(zlibbed)
```

Decoders unchanged. Streaming output is valid gzip / zlib that round-trips via this package's decoder and the reference `gzip` / `zlib` decoders.

### Decomposition (subject to brainstorm reality-check per tranche)

**Shape A (recommended pending reality-check): single-tranche-per-package, full streaming.**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 24A | swift-gzip v0.3 | `Gzip.Streaming.Encoder` (init/update/finish) + incremental CRC32 + ISIZE counter + ~12-18 tests | ~1-2 hours (pending reality-check) |
| 24B | swift-zlib v0.3 | `Zlib.Streaming.Encoder` (init/update/finish) + incremental ADLER32 + ~12-18 tests | ~1-2 hours (pending reality-check) |

**Shape B (alternate): single combined tranche** — both packages bumped together in a "codec sweep" commit.

| Tranche | Packages | Scope | Estimated calendar |
|---|---|---|---|
| 24A | swift-gzip v0.3 + swift-zlib v0.3 (paired bump) | Both streaming encoders shipped together; ~30 tests across both | ~2-3 hours combined |

Decision deferred to brainstorm. Per the multi-tranche-phase pattern (Phase 9, Phase 11), Shape A is the working assumption. Shape B reduces commit count but loses per-tranche isolation if one package's brainstorm uncovers surprises.

### Test surface (per tranche)

| Test category | Scope |
|---|---|
| Round-trip: stream encode → decode = original input | 5-8 tests (empty, single chunk, two chunks, tiny chunks, large chunk, mixed sizes) |
| Level coverage | 3-4 tests (.none, .fast, .default, .best) |
| Header/trailer correctness | 2-3 tests (gzip: FNAME + mtime + CRC32 + ISIZE; zlib: CMF/FLG check + ADLER32 BE) |
| Boundary cases | Empty stream → valid empty gzip/zlib, single byte stream |
| Error cases | Double-finish throws; update-after-finish silent no-op |
| Equivalence | Single-update streaming output decodes to same bytes as one-shot output |
| Reference parity | Optional: streaming output decodes via reference `gzip` CLI / Python `zlib` (test fixture, not asserted in CI) |

Estimated 12-18 new tests per tranche. Total Phase 24 tests added: ~24-36.

### Acceptance criteria

- v0.3.0 ships for both swift-gzip and swift-zlib with their respective `Streaming.Encoder` types.
- v0.1 + v0.2 APIs unchanged on both packages. v0.2's `Gzip.encode` and `Zlib.encode` continue to produce byte-for-byte identical output (regression test).
- Stream-encoded output decodes correctly via each package's v0.1 decoder.
- CI green on macOS + Linux, first try (continuing the 8-phase first-try streak).
- DocC includes a `### Streaming (v0.3+)` topic group on both packages.
- CHANGELOG + README updated on both packages with streaming examples.

### Out of scope (deferred to v0.4+)

- **Window carry across chunks** in the underlying DEFLATE layer (will land via swift-deflate v0.4 in Phase 26+; gzip + zlib inherit automatically).
- **Streaming decode.** Out of scope; v0.4+ candidate matched to streaming encode demand.
- **`reset()` for encoder reuse.** Match brotli + deflate v0.3 deferral.
- **Explicit flush API.** Match prior v0.3s.
- **Multi-thread streaming.**
- **Multi-member gzip streaming** (writing concatenated gzip members). v0.3 emits a single gzip member per stream.

### Migration (v0.2 → v0.3 per package)

**Additive only — non-breaking.** All v0.2 APIs unchanged on both packages:
- `Gzip.encode(_:level:filename:modificationTime:)` continues to produce byte-for-byte identical output (regression-tested).
- `Gzip.Encoder` struct unchanged.
- `Gzip.decode(_:)` unchanged from v0.1.
- `GzipError` may add 1 new case (additive).
- `Zlib.encode(_:level:)` continues to produce byte-for-byte identical output.
- `Zlib.Encoder` struct unchanged.
- `Zlib.decode(_:)` unchanged from v0.1.
- `ZlibError` may add 1 new case (additive).

Adopters opt into streaming by switching to the new `Streaming.Encoder` types.

### Risk

**Low-medium per tranche — comparable to Phase 23 deflate streaming.**

| Risk | Mitigation |
|---|---|
| swift-crc may not have incremental API; gzip 24A might need to extend swift-crc | Brainstorm 24A reality-checks swift-crc's API. If incremental missing, decide: (a) add incremental API to swift-crc (small scope; revisit RFC if so), (b) inline an incremental CRC32 in swift-gzip's Streaming.swift (~30 LOC; isolated change). |
| ADLER32 in swift-zlib is one-shot in v0.2 | Adler32.swift is 20 LOC. Trivial to add incremental `Adler32State` struct with `init` + `update` + `finalize`. ~15 LOC addition. |
| Output ratio regression vs one-shot for single-chunk feed | Test: feed single chunk, assert decoded output equals input (decoded-equality; byte-equality not guaranteed). |
| Exhaustive switches over GzipError / ZlibError in tests | Pattern reinforced: search test files for `switch.*Error` patterns when adding cases (per Gate 22 + 23 memory). |

**Risk profile is well-bounded:** both packages have small, focused v0.2 source. Streaming wraps DEFLATE (now Streaming.Encoder) + incremental checksum (small extension) + header/trailer (already in v0.2).

**Calendar estimate (~1-2 hours per tranche, ~2-4 hours total):** depends on brainstorm reality-check per tranche. Per Gate 22's canonical pattern, the brainstorm spec phase locks the estimate based on v0.2 stateful-interaction survey.

## Alternatives considered

See [Gate 23 retrospective](../docs/gates/2026-05-16-gate-23-retrospective.md) § Item 5 for the full candidate matrix. Summary:

- **swift-content-encoding v0.5 streaming wiring** — blocked on gzip + zlib streaming. Phase 25+.
- **swift-brotli v0.4 + swift-deflate v0.4 window carry** — premature; v0.3 just shipped. Phase 26+.
- **swift-oauth2-client v0.2** — Phase 25+ candidate; codec sweep completion wins this gate.
- **swift-distributed-tracing-bridge v0.3 (B3 + Jaeger)** — no concrete demand.
- **swift-idna v0.3** — rarely-hit rules; no demand.
- **RS-family JWT** — blocked on swift-crypto `_RSA` SPI.
- **Package rename swift-jwt-verify → swift-jwt** — premature.
- **Crypto-adjacent waves** — 19th consecutive rejection.

## Dependencies

- swift-gzip v0.3 will bump swift-deflate dep to >= 0.3.0 (for `Deflate.Streaming.Encoder`).
- swift-zlib v0.3 will bump swift-deflate dep to >= 0.3.0.
- swift-gzip may bump swift-crc dep if reality-check requires an incremental API addition there.
- No new direct dependencies beyond existing (swift-bytes, swift-deflate, swift-crc for gzip).

## References

- [RFC 1952](https://www.rfc-editor.org/rfc/rfc1952) — GZIP file format.
- [RFC 1950](https://www.rfc-editor.org/rfc/rfc1950) — ZLIB compressed data format.
- [Gate 23 retrospective](../docs/gates/2026-05-16-gate-23-retrospective.md) — anchor decision rationale + Phase 24 candidate survey.
- [RFC-0001](0001-conventions-and-quality-bars.md) — bare-swift conventions.
- [RFC-0014](0014-phase-9-anchor-compression-encoder-sweep.md) — Phase 9 anchor (gzip/zlib/deflate v0.2 encoders); defines the v0.2 surfaces that v0.3 augments.
- [RFC-0027](0027-phase-22-anchor-swift-brotli-v0.3-streaming-encoder.md) + [RFC-0028](0028-phase-23-anchor-swift-deflate-v0.3-streaming-encoder.md) — Phase 22 + 23 streaming-encoder templates that Phase 24 reuses.
- swift-gzip v0.2 CHANGELOG — documents v0.3 streaming deferral.
- swift-zlib v0.2 CHANGELOG — documents v0.3 streaming deferral.

## Decision

**Accepted 2026-05-16.** Phase 24 anchored on **swift-gzip v0.3 + swift-zlib v0.3 streaming encoders (2-tranche phase)**. Brainstorm + plan + execute via the bare-swift inline-execution pattern per tranche (spec → plan → coalesced inline implementation → tag → umbrella bump → memory closure). Shape A (single-tranche-per-package) is the working assumption; Shape B (combined tranche) remains a fallback if brainstorm uncovers no per-package surprises and the two tranches collapse into trivial duplication.

Per-tranche calendar estimate **deferred to brainstorm reality-check** per Gate 22's canonical pattern.

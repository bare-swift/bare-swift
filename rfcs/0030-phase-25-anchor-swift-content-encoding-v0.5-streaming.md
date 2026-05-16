# RFC-0030 — Phase 25 anchor: swift-content-encoding v0.5 (single-coding streaming wiring)

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-16 |
| Resolution | Accepted 2026-05-16 — Phase 25 plans may begin authoring. Single existing package: swift-content-encoding v0.5. |

## Summary

Anchor Phase 25 on **swift-content-encoding v0.5** — additive non-breaking minor version bump adding `ContentEncoding.Streaming.Encoder(contentEncoding:, level:)` for **single-coding HTTP `Content-Encoding` headers** (`identity`, `gzip`, `x-gzip`, `deflate`, `x-deflate`, `br`). Dispatches to the now-canonical streaming encoders in swift-gzip / swift-zlib / swift-brotli (all v0.3+). Multi-coding chains (e.g., `gzip, br`) **explicitly throw** at init because current streaming-encoder API doesn't compose mid-stream — multi-coding streaming is a Phase 26+ candidate that requires a coordinated codec-tier "drain" API.

Single-tranche; estimated calendar **determined by brainstorm reality-check** per Gate 22 canonical pattern.

## Problem

[Gate 24 retrospective](../docs/gates/2026-05-16-gate-24-retrospective.md) item 5 requires "Phase 25 anchor decision recorded as an RFC" before Phase 25 plans begin. The retrospective surveyed eight candidate waves; this RFC formalizes the choice.

**Concrete gap:** swift-content-encoding v0.4 (Phase 10D, shipped 2026-05-13) ships one-shot `ContentEncoding.decode(_:contentEncoding:)` and `ContentEncoding.encode(_:contentEncoding:level:)` for `identity`, `gzip`/`x-gzip`, `deflate`/`x-deflate`, `br`. The v0.4 CHANGELOG implicitly defers streaming to a future version (the v0.2 CHANGELOG explicitly listed "streaming encoding" as out-of-scope, and v0.3/v0.4 added br decode + br encode but kept the one-shot shape).

Phase 22-24 shipped streaming encoders in all 4 codec packages (brotli, deflate, gzip, zlib). The HTTP layer above them — content-encoding — is the natural next step.

**Why now (Phase 25):**
1. All 4 codec packages support streaming as of Phase 24 end. swift-content-encoding can finally wire them through.
2. HTTP applications wanting streaming compression (large response bodies, WebSocket frames, multipart upload) need a header-driven streaming API. v0.4 only provides one-shot, which requires buffering full response bodies.
3. Strong audience continuity from Phases 22-24 codec sweep.

**Why single-coding only (multi-coding limitation):**
Current streaming-encoder API on all 4 codec packages returns bytes **only at `finish()`**. `update(_:)` writes into the internal `BitWriter` buffer but doesn't return any bytes to the caller. This means chaining streaming encoders (e.g., gzip stream output fed to brotli stream input) requires **buffering the entire output of the first encoder** before feeding it to the second — which defeats the streaming purpose.

Two options:
- **(A) Phase 25 v0.5 single-coding streaming only.** Multi-coding chains throw early; callers fall back to v0.4 one-shot for those rare cases. Honest, clean, ships fast.
- **(B) Phase 25 + Phase 26 codec-tier "drain" API sweep.** All 4 streaming encoders gain `drain() -> Bytes` (emit accumulated bytes mid-stream, before finish). Then content-encoding v0.6 composes drain-aware chains. Larger scope; multi-package coordination.

This RFC chooses (A). Multi-coding streaming is a documented Phase 26+ candidate. Real-world HTTP traffic uses multi-coding chains ~1% of the time; single-coding streaming covers the 95%+ case.

**Why not the alternatives:**
- **swift-oauth2-client v0.2** (auth URL builder + PKCE) — strong candidate; ~2 days settle time on v0.1; valid Phase 26 candidate if v0.5 ships smoothly.
- **swift-brotli v0.4 + swift-deflate v0.4 + swift-gzip v0.4 + swift-zlib v0.4 (drain API sweep)** — premature. Phase 26+ if multi-coding streaming demand surfaces from adopters.
- **swift-distributed-tracing-bridge v0.3** (B3 + Jaeger) — no concrete demand.
- **swift-idna v0.3** — rarely-hit rules; no demand.
- **RS-family JWT** — **STILL BLOCKED** on swift-crypto's `_RSA` SPI.
- **Package rename swift-jwt-verify → swift-jwt** — premature.

## Proposal

### Anchor: swift-content-encoding v0.5 (single existing package, additive minor bump)

**swift-content-encoding v0.5** — add `ContentEncoding.Streaming.Encoder` alongside v0.4's `ContentEncoding.encode(_:contentEncoding:level:)`. v0.4's one-shot encode + decode preserved verbatim.

| Addition | Description |
|---|---|
| `ContentEncoding.Streaming` namespace enum | Public Sendable namespace. |
| `ContentEncoding.Streaming.Encoder` struct | Sendable value-type streaming encoder. |
| `init(contentEncoding:level:) throws(ContentEncodingError)` | Parse header. Single coding: dispatch to appropriate inner streaming encoder. Multi-coding: **throw** `.multipleCodingsNotStreamable` (or reuse `.unsupportedEncoding` with documented message — TBD by brainstorm). Empty header = identity passthrough. |
| `update(_ chunk: Bytes)` | Feed chunk to inner streaming encoder. For identity, accumulate internally. Empty chunk = no-op. Finished encoder = no-op. |
| `finish() throws(ContentEncodingError) -> Bytes` | Finalize inner streaming encoder. For identity, return accumulated bytes. Throws `.encoderFinished` on double-call. |
| Likely 1-2 new error cases | `.multipleCodingsNotStreamable` (or repurpose `.unsupportedEncoding`) + `.encoderFinished`. Pending brainstorm. |

### Key bytes / API shape (subject to brainstorm)

```swift
// v0.5 streaming usage (single coding):
import ContentEncoding
import Bytes

var encoder = try ContentEncoding.Streaming.Encoder(contentEncoding: "gzip", level: .default)
encoder.update(chunk1)
encoder.update(chunk2)
let compressed = try encoder.finish()
let plain = try ContentEncoding.decode(compressed, contentEncoding: "gzip")  // v0.1+ decoder

// v0.5 with multi-coding (throws):
do {
    _ = try ContentEncoding.Streaming.Encoder(contentEncoding: "gzip, br", level: .default)
} catch ContentEncodingError.multipleCodingsNotStreamable(let header) {
    // Fall back to v0.4 one-shot:
    let compressed = try ContentEncoding.encode(buffered, contentEncoding: header)
}
```

### Decomposition

**Shape A (recommended): single tranche, single-coding streaming.**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 25A | swift-content-encoding v0.5 | `ContentEncoding.Streaming.Encoder` + multi-coding-throws-at-init + ~15-20 tests | ~1-2 hours (pending reality-check) |

### Test surface

| Test category | Scope |
|---|---|
| Per-coding round-trip via v0.4 decode | identity, gzip, x-gzip, deflate, x-deflate, br — 6 tests |
| Multi-coding chain throws at init | `"gzip, br"`, `"deflate, gzip"`, `"br, deflate"` — 2-3 tests |
| Multi-chunk feeds | single chunk, two chunks, many tiny chunks, mixed sizes — 3-4 tests |
| Empty stream | per coding — handled by single test or per coding | 1-2 tests |
| Level coverage | `.default` + `.best` for at least one compressing coding — 1-2 tests |
| Error cases | double-finish, update-after-finish — 2 tests |
| Empty / whitespace header → identity passthrough | 1 test |

Estimated 15-20 new tests. Total package tests post-25A: existing-v0.4 + 15-20.

### Acceptance criteria

- v0.5.0 ships with `ContentEncoding.Streaming.Encoder` supporting single-coding streams for identity / gzip / x-gzip / deflate / x-deflate / br.
- Multi-coding headers throw at init with a clear error case + message.
- v0.4 APIs unchanged. `ContentEncoding.encode(_:contentEncoding:level:)` and `ContentEncoding.decode(_:contentEncoding:)` continue byte-equal output.
- Streaming output decodes correctly via v0.4 `ContentEncoding.decode(_:contentEncoding:)`.
- CI green on macOS + Linux, first try.
- DocC includes a `### Streaming (v0.5+)` topic group.
- CHANGELOG + README updated with streaming example and multi-coding limitation note.

### Out of scope (deferred to v0.6+ / Phase 26+)

- **Multi-coding streaming chains.** Requires a coordinated codec-tier "drain" API on swift-brotli + swift-deflate + swift-gzip + swift-zlib v0.4 (Phase 26+ candidate).
- **Streaming decode.** No streaming decoder exists in any of the 4 codec packages yet. Streaming decode is a parallel v0.4+ sweep across all codecs.
- **`reset()` for encoder reuse.** Match prior streaming-encoder deferrals.
- **Explicit flush API.** Match prior.
- **Quality / Level mapping for `br`.** swift-brotli's Quality doesn't 1:1 map to Deflate.Encoder.Level. Phase 25 streaming br ignores the `level` parameter (uses `.default` quality), matching v0.4 one-shot behavior. Future v0.6 may map.

### Migration (v0.4 → v0.5)

**Additive only — non-breaking.** All v0.4 APIs unchanged:
- `ContentEncoding.encode(_:contentEncoding:level:)` continues byte-equal output.
- `ContentEncoding.decode(_:contentEncoding:)` unchanged.
- `ContentEncodingError` adds 1-2 new cases (additive).

Adopters opt into streaming by switching from `ContentEncoding.encode` to `ContentEncoding.Streaming.Encoder` for single-coding paths.

### Risk

**Low. Pure orchestration layer over now-canonical streaming-encoder primitives in 4 codec packages.**

| Risk | Mitigation |
|---|---|
| Mapping `level` parameter to brotli's Quality is awkward | Match v0.4 behavior: ignore `level` for `br` streaming (use `.default` Quality). Documented. |
| `Brotli.Streaming.Encoder.init(quality:)` throws on invalid quality | Use `.default` (always valid). No throw. |
| Storage of inner streaming encoder requires a wrapping enum/protocol | Internal `enum InnerEncoder { case identity(Bytes); case gzip(Gzip.Streaming.Encoder); case deflate(Zlib.Streaming.Encoder); case brotli(Brotli.Streaming.Encoder) }`. Simple. |
| Identity-coding "streaming" just accumulates input | Acceptable; v0.4 identity is passthrough; v0.5 identity-streaming buffers internally and returns on finish. Documented. |
| Exhaustive-switch-over-Error in tests | Reinforced pattern; search test files for `switch.*ContentEncodingError` when adding cases. |

## Alternatives considered

See [Gate 24 retrospective](../docs/gates/2026-05-16-gate-24-retrospective.md) § Item 5 for the full candidate matrix. Summary:

- **swift-oauth2-client v0.2** — strong Phase 26 candidate.
- **swift-brotli v0.4 + swift-deflate v0.4 + swift-gzip v0.4 + swift-zlib v0.4 (drain API)** — Phase 26+ as coordinated 4-package sweep; would unblock multi-coding streaming in content-encoding v0.6.
- **swift-distributed-tracing-bridge v0.3** — no demand.
- **swift-idna v0.3** — no demand.
- **RS-family JWT** — blocked on `_RSA` SPI.
- **Package rename swift-jwt-verify → swift-jwt** — premature.
- **Crypto-adjacent** — 20th rejection.

## Dependencies

- swift-content-encoding v0.5 will bump swift-gzip + swift-zlib + swift-brotli deps to versions providing `Streaming.Encoder` (gzip 0.3+, zlib 0.3+, brotli 0.3+).
- swift-deflate dep may bump if v0.5 calls `Deflate.Streaming.Encoder` directly (currently it doesn't — deflate goes through Zlib).

## References

- [RFC 9110 § 8.4](https://www.rfc-editor.org/rfc/rfc9110#section-8.4) — Content-Encoding semantics.
- [Gate 24 retrospective](../docs/gates/2026-05-16-gate-24-retrospective.md) — anchor decision rationale + Phase 25 candidate survey + multi-coding limitation discussion.
- [RFC-0001](0001-conventions-and-quality-bars.md) — bare-swift conventions.
- [RFC-0027](0027-phase-22-anchor-swift-brotli-v0.3-streaming-encoder.md) + [RFC-0028](0028-phase-23-anchor-swift-deflate-v0.3-streaming-encoder.md) + [RFC-0029](0029-phase-24-anchor-gzip-zlib-v0.3-streaming-encoders.md) — Phase 22-24 codec-tier streaming sweep that Phase 25 wires through.
- swift-content-encoding v0.4 CHANGELOG — documents v0.5 streaming intent.

## Decision

**Accepted 2026-05-16.** Phase 25 anchored on **swift-content-encoding v0.5 single-coding streaming wiring**. Multi-coding chains explicitly throw at init; multi-coding streaming is a documented Phase 26+ candidate requiring a coordinated codec-tier drain-API sweep.

Per-tranche calendar estimate **deferred to brainstorm reality-check** per Gate 22's canonical pattern.

# RFC-0033 — Phase 28 anchor: swift-content-encoding v0.6 (multi-coding streaming)

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-17 |
| Resolution | Accepted 2026-05-17 — Phase 28 plans may begin authoring. Single existing package: swift-content-encoding v0.6. |

## Summary

Anchor Phase 28 on **swift-content-encoding v0.6** — additive non-breaking minor version bump replacing v0.5's `.multipleCodingsNotStreamable`-throws-at-init with actual multi-coding streaming via cascaded `drain()` calls on the v0.4 codec-tier streaming encoders. Completes the streaming HTTP body story (encode side) for the bare-swift ecosystem.

Single-tranche; estimated ~30-60 min (wrapper-pattern calibration). Per-tranche estimate **deferred to brainstorm reality-check** per Gate 22 canonical pattern.

## Problem

[Gate 27 retrospective](../docs/gates/2026-05-17-gate-27-retrospective.md) item 5 requires "Phase 28 anchor decision recorded as an RFC" before Phase 28 plans begin. The retrospective surveyed nine candidate waves; this RFC formalizes the choice.

**Concrete gap:** swift-content-encoding v0.5 (Phase 25A, shipped 2026-05-16) explicitly throws `ContentEncodingError.multipleCodingsNotStreamable` at init for multi-coding headers like `"gzip, br"`. The reason: v0.5's underlying streaming encoders emitted output only at `finish()`. Phase 27 (shipped 2026-05-17) added `drain() -> Bytes` to all 4 codec streaming encoders, returning byte-aligned accumulated bytes incrementally. **Multi-coding streaming is now technically feasible.**

**Why now (Phase 28):**
1. Phase 27 explicitly built drain() to unblock this.
2. Closes Phase 25's documented multi-coding limitation. Honest-scope-under-limitation pattern → completed work.
3. Completes the streaming HTTP body story (encode side): codec-tier streaming + drain + content-encoding multi-coding all done.
4. Single-tranche, tight scope (~150 LOC change + ~15 new tests).
5. Wrapper-pattern calibration (~30-60 min).

**Why not the alternatives (full survey in Gate 27 retro § Item 5):**
- **Streaming decoders sweep** — 12-20hr; Phase 29+.
- **swift-oauth2-client v0.3** — Phase 29 candidate.
- **brotli v0.5 + deflate v0.5 window carry** — premature.
- **B3+Jaeger, idna v0.3** — no demand.
- **RS-family JWT** — blocked on `_RSA`.
- **Package rename** — premature.

## Proposal

### Anchor: swift-content-encoding v0.6 (single existing package, additive minor bump)

**swift-content-encoding v0.6** — extend v0.5's `Streaming.Encoder` to handle multi-coding headers via a cascaded pipeline of inner streaming encoders, applied left-to-right at encode time per RFC 9110 § 8.4.

| Change | Description |
|---|---|
| `Streaming.Encoder.init(contentEncoding:level:)` | No longer throws `.multipleCodingsNotStreamable` for multi-coding headers. Instead, builds an ordered list of inner encoders. |
| Internal `InnerEncoder` enum | Replaced with `[InnerCoding]` array (or similar pipeline-friendly structure). |
| `update(_:)` | Cascades chunk through pipeline: feed first stage; drain first stage; feed drained bytes to second stage; ...; emit final drained bytes from last stage. |
| `finish() throws` | Cascades finish through pipeline in order: finish first stage; feed (drained+final) bytes to second stage; finish second stage; ...; return final stage's output. |
| `.multipleCodingsNotStreamable` error case | **Kept in enum** for backwards-compat with v0.5 callers that may pattern-match it. **No longer thrown** by v0.6. Documented as "deprecated but defined." |
| Dep bumps | swift-gzip 0.3 → 0.4, swift-zlib 0.3 → 0.4, swift-brotli 0.3 → 0.4 (for `drain()` access). swift-deflate dep unchanged (not directly used). |

### Pipeline semantics (locked at RFC time)

Per RFC 9110 § 8.4: `Content-Encoding` codings apply **left-to-right at encode time**, decode reverses. So `"gzip, br"` means input → gzip-encode → br-encode → output.

Pipeline build (in init):
- Parse codings (existing v0.5 `parseCodings(_:)` reuse).
- For each coding in order, create the corresponding `InnerCoding` wrapper:
  - identity → no-op
  - gzip / x-gzip → `Gzip.Streaming.Encoder`
  - deflate / x-deflate → `Zlib.Streaming.Encoder`
  - br → `Brotli.Streaming.Encoder`
- Pipeline = `[InnerCoding]` array in declaration order.

Pipeline update (in `update(_:)`):
```
input bytes → stage[0].update → stage[0].drain → stage[1].update → stage[1].drain → ... → output bytes
```

Pipeline finish (in `finish()`):
```
For each stage in order:
  stage.finish() → final bytes
  if not last stage:
    next_stage.update(final bytes)
    next_stage.drain() → intermediate bytes (collected)
  else:
    return intermediate + final
```

Or simpler equivalent: iteratively finish-then-cascade.

### Decomposition

**Shape A (recommended): single tranche, full multi-coding pipeline.**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 28A | swift-content-encoding v0.6 | Multi-coding pipeline + ~15 new tests | ~30-60 min |

### Test surface (~15 tests)

**Multi-coding round-trip:**
1. `"gzip, br"` round-trip via cascaded decode (gzip decode of br decode of compressed).
2. `"br, gzip"` round-trip.
3. `"deflate, gzip"` round-trip.
4. `"gzip, deflate, br"` (3-coding chain) round-trip.
5. `"identity, gzip"` (identity skip + gzip) — should equal pure gzip.
6. `"gzip, identity"` (gzip + identity passthrough) — should equal pure gzip.

**Multi-chunk:**
7. Multi-coding with two chunks → concatenation round-trip.
8. Empty chunk in middle of multi-coding pipeline → no-op.

**Single-coding regression:**
9. v0.5 single-coding tests still pass (gzip alone, br alone, deflate alone) — byte-equality preserved.

**Error / edge:**
10. Multi-coding with `"zstd, gzip"` (unsupported in pipeline) → throws `.unsupportedEncoding("zstd")`.
11. Multi-coding with `"gzip, zstd"` → throws `.unsupportedEncoding("zstd")`.
12. `.multipleCodingsNotStreamable` is no longer thrown for valid multi-coding chains.
13. Double-finish throws `.encoderFinished`.
14. update-after-finish silent no-op.

**Reference parity (optional):**
15. v0.6 multi-coding output `"gzip, br"` decodes correctly via `ContentEncoding.decode(_:contentEncoding: "gzip, br")` (v0.1 one-shot decoder).

### Acceptance criteria

- v0.6.0 ships with multi-coding streaming working.
- v0.5 single-coding behavior preserved (byte-equality regression-tested).
- `.multipleCodingsNotStreamable` error case still defined; no longer thrown.
- swift-gzip + swift-zlib + swift-brotli deps bumped 0.3.0 → 0.4.0.
- Output of multi-coding streaming pipeline decodes correctly via v0.1 `ContentEncoding.decode` one-shot path.
- CI green on macOS + Linux, first try.
- DocC updated with multi-coding streaming example.
- CHANGELOG + README updated.

### Out of scope (deferred to v0.7+)

- **Streaming decode.** No codec streaming decoders exist yet. Phase 29+ if adopter demand surfaces.
- **`reset()` for encoder reuse.**
- **Explicit flush API.**
- **Level → brotli Quality mapping** (still uses `.default` quality for br).
- **Decoder pipeline multi-coding** is already supported in v0.4 via reversed-coding-list dispatch; no v0.6 work needed.

### Migration (v0.5 → v0.6)

**Additive only — non-breaking.** All v0.5 APIs unchanged:
- `ContentEncoding.Streaming.Encoder(contentEncoding:level:)` no longer throws `.multipleCodingsNotStreamable` for multi-coding — instead, multi-coding now works. Callers that previously caught this error to fall back to v0.4 one-shot can remove the catch (the fall-back path is no longer needed) but the existing code continues to compile.
- `update(_:)` + `finish()` byte-for-byte preserved for single-coding (regression-tested).
- `decode(_:contentEncoding:)` unchanged from v0.1.
- `encode(_:contentEncoding:level:)` unchanged from v0.4.
- `ContentEncodingError.multipleCodingsNotStreamable` case still defined (no breaking).

### Risk

**Low. Pure orchestration extension over Phase 27's drain() API.**

| Risk | Mitigation |
|---|---|
| Cascaded drain bytes accumulate memory pressure for deeply-nested multi-coding | Multi-coding chains in real HTTP are 1-2 codings deep; not a realistic concern. Documented. |
| Pipeline order semantics confusion | RFC 9110 § 8.4 is unambiguous: left-to-right at encode time. Documented in code + tests. |
| Identity coding in pipeline middle (e.g., `"gzip, identity"`) | Identity is passthrough — input flows through unchanged. Pipeline works correctly; tested. |
| `.multipleCodingsNotStreamable` error case dead-but-kept | Documented in CHANGELOG as deprecated-but-defined for backwards-compat. |

## Alternatives considered

See [Gate 27 retrospective](../docs/gates/2026-05-17-gate-27-retrospective.md) § Item 5 for the full candidate matrix.

## Dependencies

- swift-content-encoding v0.6 bumps swift-gzip + swift-zlib + swift-brotli deps to 0.4+ (for drain() access).
- swift-deflate dep unchanged.

## References

- [RFC 9110 § 8.4](https://www.rfc-editor.org/rfc/rfc9110#section-8.4) — Content-Encoding semantics.
- [Gate 27 retrospective](../docs/gates/2026-05-17-gate-27-retrospective.md) — anchor decision rationale.
- [RFC-0030](0030-phase-25-anchor-swift-content-encoding-v0.5-streaming.md) — Phase 25 anchor with the documented multi-coding deferral.
- [RFC-0032](0032-phase-27-anchor-codec-tier-v0.4-drain-sweep.md) — Phase 27 anchor that built drain().
- swift-content-encoding v0.5 CHANGELOG — documents v0.6 deferral.

## Decision

**Accepted 2026-05-17.** Phase 28 anchored on **swift-content-encoding v0.6 multi-coding streaming**. Brainstorm + plan + execute via the bare-swift inline-execution pattern.

Per-tranche calendar estimate **deferred to brainstorm reality-check** per Gate 22's canonical pattern (~30-60 min expected; wrapper-pattern bracket).

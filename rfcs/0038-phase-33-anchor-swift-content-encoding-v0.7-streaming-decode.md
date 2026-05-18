# RFC-0038 — Phase 33 anchor: swift-content-encoding v0.7 (streaming-decode wiring)

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-18 |
| Resolution | Accepted 2026-05-18 — Phase 33 plans may begin authoring. Single existing package: swift-content-encoding v0.7. |

## Summary

Anchor Phase 33 on **swift-content-encoding v0.7** — additive non-breaking minor version bump wiring the four codec-tier streaming decoders (brotli + deflate + gzip + zlib, all v0.5) through content-encoding's streaming-decode path. Mirrors Phase 25 (single-coding streaming-encode) and Phase 28 (multi-coding streaming-encode) on the decode side. Closes the HTTP body streaming story end-to-end.

Per-tranche calendar **deferred to brainstorm reality-check** per Gate 22 canonical pattern. Shape A (single-coding only, 1 tranche) and Shape B (single + multi-coding, 2 tranches) both defensible.

## Problem

[Gate 32 retrospective](../docs/gates/2026-05-18-gate-32-retrospective.md) item 5 requires "Phase 33 anchor decision recorded as an RFC" before Phase 33 plans begin. The retrospective surveyed ten candidate waves; this RFC formalizes the choice.

**Concrete gap:** swift-content-encoding v0.6 ships streaming-encode (single-coding + multi-coding via cascaded `drain()`), but the **decode side remains one-shot only**. With Phases 30-32 shipping streaming decoders for all four codecs, the content-encoding tier can now wire streaming-decode through its `Content-Encoding` machinery.

**Why now (Phase 33):**
1. Phase 30-32 completed streaming-decode for all four codecs (deflate + gzip + zlib + brotli). The codec-tier streaming-decode sweep is COMPLETE.
2. Content-encoding's streaming-encode pattern is already established (Phase 25 `InnerEncoder` dispatch enum + Phase 28 cascaded pipeline). The decode side mirrors these.
3. HTTP middleware adopters need streaming-decode on the response path (incoming `Content-Encoding`) just as much as streaming-encode on the request path. Phase 33 delivers symmetry.
4. Closes HTTP body streaming story end-to-end. After Phase 33, both directions can stream through identity / br / deflate / gzip / zlib + compositions.
5. Validates the v0.5 buffering-wrap honest-scope choice. If content-encoding's streaming-decode wiring composes correctly over buffering-wrap codec decoders, the v0.6+ true-memory-streaming refactors become demand-driven (not capability-blocking).
6. Audience continuity from Phases 22-32 codec streaming sweeps.

**Why not the alternatives (full survey in Gate 32 retro § Item 5):**
- **swift-deflate v0.6 + swift-brotli v0.6 true memory-streaming inflate** — v0.5 just shipped; demand-driven; content-encoding v0.7 reveals whether buffering-wrap is actually a bottleneck.
- **brotli / deflate v0.6 window carry** — ratio improvement; not blocking.
- **swift-oauth2-client v0.4** — v0.3 shipped Phase 29; no concrete adopter demand.
- **B3 + Jaeger, idna v0.3** — long-running correct-deferrals; no demand (>13 each).
- **RS-family JWT** — STILL BLOCKED on swift-crypto `_RSA` SPI.
- **Package rename** — premature.

## Proposal

### Anchor: swift-content-encoding v0.7 (single existing package, additive minor bump)

**swift-content-encoding v0.7** — wire the four codec streaming decoders through the content-encoding tier. v0.1-v0.6 APIs unchanged.

| Addition | Description |
|---|---|
| `ContentEncoding.Streaming.Decoder` (or equivalent type per brainstorm) | Sendable value-type streaming decoder. Mirrors `ContentEncoding.Streaming.Encoder`'s shape from v0.5/v0.6: init(coding:) / update(_:) / finish() throws -> Bytes. |
| `InnerDecoder` dispatch enum (internal) | Mirrors v0.5's `InnerEncoder` dispatch enum. Selects between brotli / deflate / gzip / zlib streaming decoders + identity passthrough based on coding. |
| Multi-coding decode path (Shape B only) | Cascaded streaming-decode pipeline mirror of v0.6's cascaded-encode pipeline. Handles `Content-Encoding: gzip, br` (chained codings, RFC 9110 § 8.4) by streaming through inner→outer decoders. |
| New error case(s) | Likely `ContentEncodingError.decoderFinished` (mirror encoder pattern) — TBD by brainstorm. |

### Brainstorm decision: Shape A (single-coding only) vs Shape B (single + multi-coding)

Per Phase 28's brainstorm-empowered-by-RFC scope simplification precedent, brainstorm chooses between:

- **Shape A: single-coding streaming-decode only.** One tranche. Single-coding (the 95%+ case in practice) ships in Phase 33A; multi-coding deferred to Phase 34. ~1-2 hours.
- **Shape B: single + multi-coding in two tranches.** Phase 33A single-coding + Phase 33B multi-coding cascaded pipeline. Closes the symmetric end-to-end story in one phase. ~2-4 hours total.

Phase 28's precedent suggests Shape B is feasible if reality-check confirms the cascaded-pipeline pattern mirrors cleanly to decode. Phase 28 shipped multi-coding-encode in a single tranche over an already-shipped single-coding-encode (Phase 25), so this would be a similar one-shot lift.

### Test surface (~12-15 tests per tranche)

| Test category | Scope |
|---|---|
| Single-coding round-trip via v0.5 encoder | For each coding (identity, br, deflate, gzip, zlib): encode then decode → bytes match | 5 tests |
| Multi-chunk decode | Split a v0.5-encoded stream into 2/3/many chunks → stream-decode → bytes match | 2-3 tests |
| Tiny 1-byte chunks | Pathological chunk size; round-trip | 1 test |
| Error cases | Truncated input throws; double-finish throws decoderFinished; update-after-finish no-op | 3 tests |
| Edge cases | Empty stream; single-byte payload; identity passthrough | 2-3 tests |
| (Shape B only) Multi-coding | `gzip, br` chained codings: encode-then-decode round-trip | 3-4 additional tests |

Estimated 12-15 tests for Shape A; +3-4 for Shape B.

### Acceptance criteria

- v0.7.0 ships with content-encoding streaming-decode.
- v0.1-v0.6 APIs unchanged. v0.6 streaming-encode + multi-coding cascaded pipeline continue byte-for-byte.
- Stream-decoded output equals one-shot decode output for all test cases.
- CI green on macOS + Linux, first try.
- DocC includes `### Streaming decode (v0.7+)` topic group.
- CHANGELOG + DocC updated with streaming-decode example.
- **DocC discipline:** cross-package symbol refs (to Brotli, Deflate, Gzip, Zlib modules) use **single-backtick** per first-line MEMORY note. Re-emphasized after Phase 31A's CI break; held in Phase 32A.
- Codec dependency bumps if needed (likely already at 0.5 for all four; verify and bump in lockstep).

### Out of scope (deferred to v0.8+ or downstream phases)

- True memory-streaming through buffering-wrap codec decoders (blocked on codec v0.6+ state-machine refactors).
- New coding additions beyond identity / br / deflate / gzip / zlib.
- HTTP-layer integration sweeps (separate package work).

### Migration (v0.6 → v0.7)

**Additive only — non-breaking.** All v0.1-v0.6 APIs unchanged. New `Streaming.Decoder` (or equivalent) type added.

### Risk

**Low-medium per brainstorm-locked shape:**
- **Shape A (single-coding only)**: Low risk. Pure mirror of Phase 25 `InnerEncoder` dispatch. ~1-2 hours.
- **Shape B (single + multi-coding)**: Low-medium risk. Multi-coding decode cascade is a mirror of Phase 28's encode cascade; well-understood pattern. ~2-4 hours total.

| Risk | Mitigation |
|---|---|
| Buffering-wrap codec decoders compose poorly with content-encoding streaming | If discovered, surface as honest-scope-under-limitation note. Adopters get a streaming API surface; true memory-streaming awaits codec v0.6+. Matches the 5-instance pattern. |
| Cross-package DocC ref slip on codec symbols | Phase 33 docstrings will reference codec types frequently. Single-backtick discipline mandatory. Apply memory note as checklist (Gate 31 lesson). |
| Multi-coding decode cascade ordering bugs | Decode is inverse-order from encode (last-applied coding decodes first). Reality-check this in brainstorm. |

## Alternatives considered

See [Gate 32 retrospective](../docs/gates/2026-05-18-gate-32-retrospective.md) § Item 5 for the full candidate matrix.

## Dependencies

- swift-brotli ≥ 0.5 (Brotli.Streaming.Decoder; shipped Phase 32A).
- swift-deflate ≥ 0.5 (Deflate.Streaming.Decoder; shipped Phase 30A).
- swift-gzip ≥ 0.5 (Gzip.Streaming.Decoder; shipped Phase 31A).
- swift-zlib ≥ 0.5 (Zlib.Streaming.Decoder; shipped Phase 31B).
- No new external dependencies.

## References

- [RFC 9110 § 8.4](https://www.rfc-editor.org/rfc/rfc9110#section-8.4) — HTTP `Content-Encoding` semantics.
- [Gate 32 retrospective](../docs/gates/2026-05-18-gate-32-retrospective.md) — anchor decision rationale.
- [RFC-0031](0031-phase-25-anchor-swift-content-encoding-v0.5-streaming.md) — Phase 25 (content-encoding v0.5 single-coding streaming-encode); the `InnerEncoder` dispatch precedent.
- [RFC-0034](0034-phase-28-anchor-swift-content-encoding-v0.6-multi-coding-streaming.md) — Phase 28 (content-encoding v0.6 multi-coding cascaded-encode pipeline); the cascaded-pipeline precedent.
- [RFC-0037](0037-phase-32-anchor-swift-brotli-v0.5-streaming-inflate.md) — Phase 32 (brotli v0.5; final codec streaming decoder).
- [feedback_docc_cross_package](MEMORY first-line) — single-backtick for cross-package DocC refs.

## Decision

**Accepted 2026-05-18.** Phase 33 anchored on **swift-content-encoding v0.7 streaming-decode wiring**. Brainstorm + plan + execute via the bare-swift inline-execution pattern. Per-tranche calendar estimate **deferred to brainstorm reality-check** per Gate 22's canonical pattern. Shape A vs Shape B decided by brainstorm.

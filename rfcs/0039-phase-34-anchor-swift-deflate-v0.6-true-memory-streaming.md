# RFC-0039 — Phase 34 anchor: swift-deflate v0.6 (true memory-streaming inflate)

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-18 |
| Resolution | Accepted 2026-05-18 — Phase 34 plans may begin authoring. Single existing package: swift-deflate v0.6 (potentially expanded to coordinated gzip + zlib v0.6 per brainstorm reality-check). |

## Summary

Anchor Phase 34 on **swift-deflate v0.6** — refactor the internal Inflater into a state-machine that yields decompressed bytes incrementally rather than buffering all compressed input until `finish()`. The public `Deflate.Streaming.Decoder` API surface (init/update/finish) remains unchanged — adopters pay zero migration cost. Resolves the 6-instance honest-scope-under-limitation pattern at the codec-tier foundation; unlocks downstream gzip/zlib v0.6 inheritance and content-encoding v0.8 propagation.

Single-tranche default (Shape A); per-tranche calendar **deferred to brainstorm reality-check** per Gate 22 canonical pattern. State-machine-work calibration bucket is 3-5 hr per Phase 29.

## Problem

[Gate 33 retrospective](../docs/gates/2026-05-18-gate-33-retrospective.md) item 5 requires "Phase 34 anchor decision recorded as an RFC" before Phase 34 plans begin. The retrospective surveyed ten candidate waves; this RFC formalizes the choice.

**Concrete gap:** swift-deflate v0.5 shipped `Deflate.Streaming.Decoder` with honest-scope-under-limitation: the implementation buffers all compressed input internally and runs `Deflate.decode(_:)` one-shot at `finish()`. True memory-streaming inflate requires a state-machine refactor of the internal `Inflater` (yielded bytes per `update(_:)` call rather than at `finish()` only).

The streaming-symmetric API surface (init/update/finish) was deliberately shipped ahead of true streaming so downstream consumers could adopt without waiting. Phase 34 validates that bet — the refactor must preserve the public shape byte-for-byte.

**Why now (Phase 34):**
1. HTTP body streaming story is COMPLETE end-to-end (Phase 33). Phase 34+ shifts from capability-blocking work to optimization.
2. 6-instance honest-scope-under-limitation pattern (Phases 25 → 28 → 30 → 31 → 32 → 33) is fully canonical. Phase 34 is the first opportunity to **resolve** a long-standing instance rather than add another.
3. Resolving deflate v0.6 unlocks gzip v0.6 + zlib v0.6 inheritance automatically (they wrap Deflate.Streaming.Decoder per Phase 31 pattern); content-encoding v0.8 inherits via dep bump.
4. Validates the streaming-symmetric-API-surface-ahead-of-true-streaming sub-pattern from Phase 32. If the refactor preserves the public shape, the pattern is canonical for future versioning decisions.
5. Audience continuity from Phases 22-33 codec streaming sweeps.
6. Brotli v0.6 (true memory-streaming) is higher risk than deflate (more complex internals). Deflate first; brotli follows in Phase 35+ if Phase 34 lessons apply cleanly.

**Why not the alternatives (full survey in Gate 33 retro § Item 5):**
- **swift-brotli v0.6 true memory-streaming inflate** — higher risk than deflate; defer to Phase 35+.
- **swift-oauth2-client v0.4** — v0.3 shipped Phase 29; no concrete adopter demand surfaced.
- **brotli / deflate v0.6 window carry** — ratio improvement; not blocking; can ride with v0.6 if scope-tractable.
- **B3 + Jaeger, idna v0.3** — long-running correct-deferrals; no demand.
- **RS-family JWT** — STILL BLOCKED on swift-crypto `_RSA` SPI.
- **Crypto-adjacent** — 29th rejection.
- **Package rename** — premature.

## Proposal

### Anchor: swift-deflate v0.6 (single existing package, additive minor bump)

**swift-deflate v0.6** — refactor internal `Inflater` to a state-machine yielding bytes incrementally. v0.1-v0.5 public APIs unchanged.

| Change | Description |
|---|---|
| Internal Inflater state-machine refactor | Loop variables + output buffer position become struct fields; the inflate loop becomes a stepping function that consumes available input and yields available output until either input is exhausted (await more) or output cap reached (yield + resume). Huffman code tables are computed once at first chunk. |
| `Deflate.Streaming.Decoder.update(_:)` semantics | Yields decoded bytes incrementally per chunk. Replaces v0.5's buffering-wrap behavior. The yielded bytes accumulate in an internal output buffer drained at finish(). |
| `Deflate.Streaming.Decoder` public API surface | **Unchanged.** init() / update(_ chunk: Bytes) / finish() throws(DeflateError) -> Bytes. Adopters require no migration. |
| Honest-scope CHANGELOG note (from v0.5) | Removed in v0.6 — limitation resolved. |
| Tests | All v0.5 tests must continue to pass. Add tests that verify incremental yield (e.g., feed compressed bytes in chunks; assert intermediate update calls produce non-empty buffered output for inputs that v0.5 would have held empty). |

### Brainstorm decision: Shape A (deflate only) vs Shape B (deflate + gzip + zlib coordinated)

Per Phase 30 + 32 + 33 brainstorm-empowered-by-RFC scope simplification precedent (3-instance canonical), brainstorm chooses between:

- **Shape A (recommended): single tranche.** swift-deflate v0.6 only. gzip + zlib v0.5 continue to inherit via wrap until they bump deflate dep to 0.6 in a later phase. ~3-5 hours. Lowest coordination cost.
- **Shape B (alternate): 2-3 tranche cross-package phase.** swift-deflate v0.6 + swift-gzip v0.6 + swift-zlib v0.6 simultaneously. The gzip/zlib tranches are pure dep-bumps with no logic changes (inheritance does the work). ~3-5 hr + 30-60 min × 2 = ~5-7 hr total. Closes the deflate-family v0.6 sweep in one phase.

Phase 31's precedent for gzip + zlib v0.5 (2-tranche coordinated) supports Shape B's feasibility. Phase 34's brainstorm reality-check decides based on whether the deflate refactor introduces any behavioral subtlety that gzip/zlib wraps need to re-test (e.g., new error cases, new yield-timing properties).

### Test surface

| Test category | Scope | Estimated count |
|---|---|---|
| All v0.5 streaming tests | Must continue to pass | regression |
| Incremental yield | Feed compressed bytes in chunks; assert intermediate update calls produce non-empty buffered output (where v0.5 would have held empty) | 3-5 tests |
| Large input + small output cap | Verify yield-and-resume semantics work for input larger than internal output buffer cap | 1-2 tests |
| State preservation across chunks | Verify Huffman tables / bit position / output buffer state survive chunk boundaries | 2-3 tests |
| Error mid-stream | Truncated input partway through a chunk; assert finish() throws appropriately | 1-2 tests |
| Equivalence with one-shot | Stream-decode chunk-by-chunk equals `Deflate.decode(_:)` one-shot | 1 test |

Estimated 8-13 new tests. Total package tests post-34A: existing v0.5 + ~10 = ~10% increase.

### Acceptance criteria

- v0.6.0 ships with state-machine inflate.
- v0.1-v0.5 APIs unchanged. `Deflate.Streaming.Decoder` shape + behavior at finish() byte-for-byte preserved.
- Incremental yield observable via at least one test that v0.5 would fail.
- All existing v0.5 tests continue to pass.
- CI green on macOS + Linux, first try.
- DocC includes update to streaming-decode topic group noting v0.6's true memory-streaming.
- CHANGELOG v0.6.0 entry documents the implementation upgrade + records that honest-scope-under-limitation pattern is now **resolved** at the codec-tier foundation.
- README updated with streaming-decode example (or deferred per orchestration-sweep pattern — Shape A allows deferral; Shape B's deflate-family sweep arguably warrants README updates).
- **DocC discipline:** any cross-package symbol refs use **single-backtick** per first-line MEMORY note.

### Out of scope (deferred to v0.7+ or later phases)

- gzip + zlib v0.6 dep bumps (if Shape A chosen; deferred to Phase 35+ or rolled into Phase 34 as Shape B).
- swift-content-encoding v0.8 dep bump (Phase 35+ candidate after deflate-family v0.6 settles).
- swift-brotli v0.6 (Phase 35+ candidate).
- Window-carry ratio improvements (orthogonal; Phase 35+ candidate).
- `reset()` for decoder reuse.

### Migration (v0.5 → v0.6)

**Additive only — non-breaking.** All v0.1-v0.5 APIs unchanged. The implementation upgrade is internal; adopters require no code changes.

### Risk

**High per brainstorm-locked shape:**
- **Shape A (deflate only)**: High risk on the refactor itself (state-machine work on a non-trivial decompressor); low risk on coordination. ~3-5 hr.
- **Shape B (deflate + gzip + zlib coordinated)**: Same refactor risk + low-medium coordination risk. ~5-7 hr.

| Risk | Mitigation |
|---|---|
| State-machine refactor introduces regression vs v0.5 buffering-wrap | Strict regression suite — every v0.5 test must pass byte-for-byte. New tests verify incremental yield without changing finish() output. |
| Internal Huffman/bit-position state not cleanly factorable into a state-machine | Brainstorm reality-check before committing. If refactor is intractable, downgrade to Shape A scope reduction (partial refactor) or defer to Phase 35+. |
| Cross-package DocC ref slip on codec module names | Single-backtick discipline mandatory. Apply memory note as checklist (Gate 31 lesson). |
| Behavioral subtlety surfaces for gzip/zlib wraps | Shape B mitigates by testing all three immediately; Shape A leaves the question to Phase 35+. |

## Alternatives considered

See [Gate 33 retrospective](../docs/gates/2026-05-18-gate-33-retrospective.md) § Item 5 for the full candidate matrix.

## Dependencies

- No new external dependencies. swift-deflate's existing deps unchanged.

## References

- [RFC 1951](https://www.rfc-editor.org/rfc/rfc1951) — DEFLATE compressed data format.
- [Gate 33 retrospective](../docs/gates/2026-05-18-gate-33-retrospective.md) — anchor decision rationale.
- [RFC-0035](0035-phase-30-anchor-swift-deflate-v0.5-streaming-inflate.md) — Phase 30 (deflate v0.5 buffering-wrap streaming decode); the honest-scope-under-limitation deferral being resolved here.
- [RFC-0036](0036-phase-31-anchor-gzip-zlib-v0.5-streaming-inflate.md) — Phase 31 (gzip + zlib v0.5 wrapper-pattern); the inheritance-via-dep-bump precedent.
- [RFC-0037](0037-phase-32-anchor-swift-brotli-v0.5-streaming-inflate.md) — Phase 32 (brotli v0.5 buffering-wrap); a parallel honest-scope instance to resolve in Phase 35+.
- [RFC-0038](0038-phase-33-anchor-swift-content-encoding-v0.7-streaming-decode.md) — Phase 33 (content-encoding v0.7); downstream consumer whose v0.8 will inherit Phase 34's refactor.
- [feedback_docc_cross_package](MEMORY first-line) — single-backtick for cross-package DocC refs.

## Decision

**Accepted 2026-05-18.** Phase 34 anchored on **swift-deflate v0.6 true memory-streaming inflate**. Brainstorm + plan + execute via the bare-swift inline-execution pattern. Per-tranche calendar estimate **deferred to brainstorm reality-check** per Gate 22's canonical pattern. Shape A vs Shape B decided by brainstorm.

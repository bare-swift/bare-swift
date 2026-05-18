# RFC-0041 — Phase 36 anchor: swift-brotli v0.6 (true memory-streaming inflate)

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-18 |
| Resolution | Accepted 2026-05-18 — Phase 36 plans may begin authoring. Primary package: swift-brotli v0.6. Optional Tranche 36B: swift-content-encoding v0.9 (uniform propagation; rides on 36A). |

## Summary

Anchor Phase 36 on **swift-brotli v0.6** — refactor the internal `Decoder` into a state-machine that yields decompressed bytes incrementally rather than buffering all compressed input until `finish()`. The public `Brotli.Streaming.Decoder` API surface (init/update/finish) remains unchanged — adopters pay zero migration cost. **Closes the codec-tier true-memory-streaming story entirely** (deflate v0.6 + gzip v0.6 + zlib v0.6 already shipped via Phase 34 + 35); resolves the last codec-tier honest-scope-under-limitation instance.

Optional Tranche 36B unblocked: swift-content-encoding v0.9 (pure dep bump; replaces Phase 35C's partial-propagation v0.8 with uniform propagation). Brainstorm decides whether to bundle 36B same-phase or defer to Phase 37.

Per-tranche calendar **deferred to brainstorm reality-check** per Gate 22 canonical pattern. State-machine-work calibration bucket is 4-6 hr for brotli's complexity (deflate v0.6 came in at 2.5 hr per Phase 34; brotli Decoder is more complex).

## Problem

[Gate 35 retrospective](../docs/gates/2026-05-18-gate-35-retrospective.md) item 5 requires "Phase 36 anchor decision recorded as an RFC" before Phase 36 plans begin. The retrospective surveyed nine candidate waves; this RFC formalizes the choice.

**Concrete gap:** swift-brotli v0.5 shipped `Brotli.Streaming.Decoder` with honest-scope-under-limitation: the implementation buffers all compressed input internally and runs `Brotli.decode(_:)` one-shot at `finish()`. True memory-streaming inflate requires a state-machine refactor of the internal `Decoder` plus supporting helpers (`MetaBlockHeader`, `OutputBuffer`, `ContextMap`, `PrefixCode`).

The streaming-symmetric API surface (init/update/finish) was deliberately shipped ahead of true streaming in Phase 32 so downstream consumers could adopt without waiting. Phase 36 validates that bet at higher state-machine complexity than Phase 34's deflate work — the refactor must preserve the public shape byte-for-byte.

**Why now (Phase 36):**
1. Phase 34 + 35 closed the deflate-family true-memory-streaming arc. Brotli is the last codec without true memory-streaming. Phase 36 closes the codec-tier story.
2. Resolves the last codec-tier honest-scope-under-limitation instance. After Phase 36, the codec-tier has no remaining buffering-wrap deferrals.
3. Unlocks content-encoding v0.9 uniform propagation (replacing Phase 35C's partial-propagation v0.8). Adopters using multi-coding chains that include `br` get true-memory-streaming end-to-end.
4. Validates the snapshot-and-restore pattern at higher complexity. Phase 34 codified the technique on deflate; Phase 36 exercises it on brotli's metablock + multi-sub-decoder structure. If the pattern generalizes cleanly, it's a strong general-purpose-streaming-codec-refactor validation.
5. Audience continuity from Phases 22-35 codec streaming sweeps.
6. State-machine-work calibration bucket from Phase 29 (3-5 hr) was extended to 4-6 hr for brotli; this phase tests that bracket.

**Why not the alternatives (full survey in Gate 35 retro § Item 5):**
- **swift-content-encoding v0.9 alone** — blocked on Phase 36's brotli v0.6 (could ride as Tranche 36B).
- **swift-oauth2-client v0.4** — v0.3 shipped Phase 29; no concrete adopter demand.
- **brotli / deflate v0.6 window carry** — ratio improvement; not blocking; can ride later.
- **B3 + Jaeger, idna v0.3** — long-running correct-deferrals; no demand.
- **RS-family JWT** — STILL BLOCKED on swift-crypto `_RSA` SPI.
- **Crypto-adjacent** — 29th rejection.
- **Package rename** — premature.

## Proposal

### Anchor: swift-brotli v0.6 (existing package, additive minor bump)

**swift-brotli v0.6** — refactor internal `Decoder` to a state-machine yielding bytes incrementally. v0.1-v0.5 public APIs unchanged.

| Change | Description |
|---|---|
| Internal Decoder state-machine refactor | Decoder's loop variables + output buffer position + metablock state + Huffman tables + context maps become struct fields; the decode loop becomes a stepping function that consumes available input and yields available output until either input is exhausted (await more) or output cap reached. |
| Supporting helpers refactored as needed | `MetaBlockHeader`, `OutputBuffer`, `ContextMap`, `PrefixCode` may need state-preservation methods (snapshot/restore or equivalent). Brainstorm reality-check identifies the minimum viable subset. |
| `Brotli.Streaming.Decoder.update(_:)` semantics | Yields decoded bytes incrementally per chunk. Replaces v0.5's buffering-wrap behavior. |
| `Brotli.Streaming.Decoder` public API surface | **Unchanged.** init() / update(_ chunk: Bytes) / finish() throws(BrotliError) -> Bytes. Adopters require no migration. |
| Honest-scope CHANGELOG note (from v0.5) | Removed in v0.6 — limitation resolved. |
| Tests | All v0.5 tests must continue to pass. Add tests that verify incremental yield (analogous to Phase 34's 5 v0.6 tests). |

### Phase 34 sub-patterns to apply

Per Gate 34 retrospective, four sub-patterns are available for this work:

1. **Snapshot-and-restore for resumable state machines.** Generic technique: inner data source (BitReader in deflate; the analogous brotli bit reader) exposes a `Snapshot` value + `snapshot()` / `restore(_:)` methods; the state machine snapshots before truncation-risky reads and restores on `.truncated`.
2. **Real-error-capture-during-update + rethrow-at-finish.** API-compat technique: capture real decode errors in a `pendingError` field during `update(_:)`; surface at `finish()`. Preserves "update doesn't throw / finish throws" contract.
3. **Two-track coexistence.** Keep the existing one-shot `Decoder` path used by `Brotli.decode(_:)` alongside the new `StreamingDecoder` path used by `Brotli.Streaming.Decoder`. Both maintained until performance + behavior parity is proven across adopters.
4. **Early-out for trivially-completable state-machine cases.** Check terminal conditions before resource-availability checks. Surfaced by Phase 34's zero-length-stored-block bug.

### Brainstorm decision: Shape A (brotli only) vs Shape B (brotli + content-encoding v0.9)

Per 4-instance brainstorm-empowered-by-RFC scope simplification pattern:

- **Shape A: brotli v0.6 only.** Single tranche. content-encoding v0.9 deferred to Phase 37+ as a pure dep-bump. Focuses Phase 36 on the high-risk state-machine refactor. ~4-6 hr.
- **Shape B: brotli v0.6 + content-encoding v0.9 (2 tranches).** 36A brotli v0.6 state-machine refactor + 36B content-encoding v0.9 pure dep-bump same-day. Closes the codec-tier + content-encoding stories in one phase. Phase 35's 3-tranche same-day shipping precedent supports feasibility. ~4-6 hr + 15-20 min = ~4.5-6.5 hr total.

Brainstorm reality-checks based on:
- Whether 36A's refactor goes smoothly (if it does, 36B is a 15-20 min victory lap).
- Whether bundling 36B same-phase risks scope creep on 36A.

Shape B is the natural codec-tier story closer; Shape A is the safer scope discipline.

### Test surface

| Test category | Scope | Estimated count |
|---|---|---|
| All v0.5 streaming tests | Must continue to pass | regression |
| Incremental yield | Feed compressed bytes in chunks; assert intermediate update calls produce non-empty buffered output | 3-5 tests |
| Large input + small output cap | Verify yield-and-resume semantics work for input larger than internal output buffer cap | 1-2 tests |
| State preservation across chunks | Verify metablock state / Huffman tables / context maps survive chunk boundaries | 2-3 tests |
| Error mid-stream | Truncated input partway through a chunk; assert finish() throws appropriately | 1-2 tests |
| Equivalence with one-shot | Stream-decode chunk-by-chunk equals `Brotli.decode(_:)` one-shot | 1 test |

Estimated 8-13 new tests (matching Phase 34's RFC bracket). Phase 34 actual was 5; Phase 36 may also come in below bracket.

### Acceptance criteria

- v0.6.0 ships with state-machine inflate.
- v0.1-v0.5 APIs unchanged. `Brotli.Streaming.Decoder` shape + behavior at finish() byte-for-byte preserved.
- Incremental yield observable via at least one test that v0.5 would fail.
- All existing v0.5 tests continue to pass.
- CI green on macOS + Linux, first try.
- DocC includes update to streaming-decode topic group noting v0.6's true memory-streaming.
- CHANGELOG v0.6.0 entry documents the implementation upgrade + records that honest-scope-under-limitation is now resolved.
- **DocC discipline:** cross-package symbol refs (rare for brotli; mostly in-package work) use **single-backtick** per first-line MEMORY note.

### Shape B Tranche 36B scope (if chosen)

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 36B | swift-content-encoding v0.9 | Dep bump swift-brotli 0.5 → 0.6; verify existing tests pass; CHANGELOG v0.9.0 note: uniform propagation (replaces v0.8's partial-propagation acknowledgment); update tagline | ~15-20 min |

### Out of scope (deferred to v0.7+ or later phases)

- Brotli window-carry across chunks (separate work; ratio improvement; Phase 37+ candidate).
- `reset()` for decoder reuse.
- Unifying one-shot + streaming inflate paths in brotli (two-track coexistence is intentional initially).

### Migration (v0.5 → v0.6)

**Additive only — non-breaking.** All v0.1-v0.5 APIs unchanged. The implementation upgrade is internal; adopters require no code changes.

### Risk

**High per brainstorm-locked shape:**
- **Shape A (brotli only)**: High risk on the refactor itself (state-machine work on a complex decoder with multiple helper structs); low risk on coordination. ~4-6 hr.
- **Shape B (brotli + content-encoding v0.9)**: Same refactor risk + low coordination risk. ~4.5-6.5 hr.

| Risk | Mitigation |
|---|---|
| State-machine refactor introduces regression vs v0.5 buffering-wrap | Strict regression suite — every v0.5 test must pass byte-for-byte. New tests verify incremental yield without changing finish() output. |
| Brotli helper structs (MetaBlockHeader / OutputBuffer / ContextMap / PrefixCode) resist clean state-preservation factoring | Brainstorm reality-check before committing. If refactor is intractable, downgrade scope (partial refactor; defer parts to v0.7+) or defer brotli v0.6 to Phase 37+. |
| Phase 34's snapshot-and-restore pattern doesn't generalize to brotli's bit reader | Apply with adaptation; document any divergence as a new pattern variant. |
| Shape B risks scope creep on 36A | Shape A is the safer default. Choose Shape B only if 36A finishes early. |

## Alternatives considered

See [Gate 35 retrospective](../docs/gates/2026-05-18-gate-35-retrospective.md) § Item 5 for the full candidate matrix.

## Dependencies

- No new external dependencies. swift-brotli's existing deps unchanged.

## References

- [RFC 7932](https://www.rfc-editor.org/rfc/rfc7932) — Brotli compressed data format.
- [Gate 35 retrospective](../docs/gates/2026-05-18-gate-35-retrospective.md) — anchor decision rationale.
- [RFC-0037](0037-phase-32-anchor-swift-brotli-v0.5-streaming-inflate.md) — Phase 32 (brotli v0.5 buffering-wrap); the honest-scope-under-limitation deferral being resolved here.
- [RFC-0039](0039-phase-34-anchor-swift-deflate-v0.6-true-memory-streaming.md) — Phase 34 (deflate v0.6 true memory-streaming); the precedent + four sub-patterns being applied here.
- [RFC-0040](0040-phase-35-anchor-downstream-propagation-sweep.md) — Phase 35 (downstream propagation); the wrap-pattern inheritance to be replicated in Tranche 36B (or Phase 37).
- [feedback_docc_cross_package](MEMORY first-line) — single-backtick for cross-package DocC refs.

## Decision

**Accepted 2026-05-18.** Phase 36 anchored on **swift-brotli v0.6 true memory-streaming inflate**. Brainstorm + plan + execute via the bare-swift inline-execution pattern. Per-tranche calendar estimate **deferred to brainstorm reality-check** per Gate 22's canonical pattern. Shape A vs Shape B decided by brainstorm.

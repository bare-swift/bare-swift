# RFC-0042 — Phase 37 anchor: swift-content-encoding v0.9 (uniform brotli propagation)

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-19 |
| Resolution | Accepted 2026-05-19 — Phase 37 plans may begin authoring. Single existing package: swift-content-encoding v0.9. |

## Summary

Anchor Phase 37 on **swift-content-encoding v0.9** — pure dep bump replacing v0.8's partial-propagation acknowledgment (deflate/gzip/zlib at 0.6, brotli pinned at 0.5) with uniform true-memory-streaming via swift-brotli 0.5 → 0.6. **Closes the codec-tier true-memory-streaming story end-to-end**; **resolves the final honest-scope-under-limitation instance** (CE v0.8's brotli-stage buffering); **caps the 7-phase arc (Phase 30 → 36) cleanly**.

Per-tranche calendar **deferred to brainstorm reality-check** per Gate 22 canonical pattern. Wrapper-pattern + pure-dep-bump sub-bucket calibration is ~10-20 min per Phase 35 precedent.

## Problem

[Gate 36 retrospective](../docs/gates/2026-05-19-gate-36-retrospective.md) item 5 requires "Phase 37 anchor decision recorded as an RFC" before Phase 37 plans begin. The retrospective surveyed eight candidate waves; this RFC formalizes the choice.

**Concrete gap:** swift-content-encoding v0.8 shipped uniform true-memory-streaming for deflate/gzip/zlib coding paths but pinned swift-brotli at 0.5 (buffering-wrap) because brotli v0.6 hadn't shipped at that point. The v0.8 CHANGELOG documents this as partial-propagation-acknowledgment — multi-coding chains containing `br` still buffer through the brotli stage. Phase 36 shipped brotli v0.6 (true memory-streaming); v0.9 bumps the brotli dep to 0.6 and replaces the partial-propagation note with uniform-propagation.

**Why now (Phase 37):**
1. Phase 36 shipped brotli v0.6; the dep is now available.
2. Closes the codec-tier true-memory-streaming story end-to-end (Phase 30 → 31 → 32 → 33 → 34 → 35 → 36 → 37). The 7-phase arc — the most ambitious multi-phase project in bare-swift — finally fully concludes.
3. Resolves the final honest-scope-under-limitation instance (CE v0.8 brotli-stage buffering).
4. Lowest risk on the candidate queue — pure dep bump; no logic changes; regression on existing tests sufficient. Phase 35's wrapper-pattern + pure-dep-bump sub-bucket calibration (~10-20 min) applies.
5. HTTP middleware adopters using content-encoding with `br`-containing multi-coding chains get true-memory-streaming end-to-end for free.
6. Validates Phase 36 under real wrapper-pattern use — if existing tests pass unchanged, brotli v0.6's state-machine refactor is confirmed correct under composition.

**Why not the alternatives (full survey in Gate 36 retro § Item 5):**
- **swift-oauth2-client v0.4** — v0.3 shipped Phase 29; significant settle time but no concrete adopter demand surfaced.
- **brotli / deflate v0.7 window carry** — ratio improvement; not blocking.
- **B3 + Jaeger, idna v0.3** — long-running correct-deferrals; no demand.
- **RS-family JWT** — STILL BLOCKED on swift-crypto `_RSA` SPI.
- **Crypto-adjacent** — 29th rejection.
- **Package rename** — premature.

## Proposal

### Anchor: swift-content-encoding v0.9 (single existing package, additive minor bump)

**swift-content-encoding v0.9** — pure dep bump swift-brotli 0.5 → 0.6; replaces v0.8's partial-propagation note with uniform-propagation. v0.1-v0.8 public APIs unchanged.

| Change | Description |
|---|---|
| Package.swift | swift-brotli dep floor: 0.5.0 → 0.6.0. (deflate/gzip/zlib already at 0.6 from Phase 35C.) |
| CHANGELOG v0.9.0 entry | Document uniform-propagation. Replace v0.8's partial-propagation acknowledgment with: "brotli now true-memory-streams via swift-brotli 0.6 (Phase 36); all coding chains — single or multi-coding, with or without `br` — now run true-memory-streaming end-to-end. The v0.8 partial-propagation acknowledgment is retired." |
| DocC tagline | Update to mention v0.9 uniform propagation. |
| Tests | All v0.8 tests (91) must continue to pass. Optionally add 1-2 tests exercising end-to-end brotli + multi-coding incremental yield through CE v0.9. |

### Test surface

| Test category | Scope | Estimated count |
|---|---|---|
| All v0.8 streaming tests | Must continue to pass (regression) | regression |
| Optional: end-to-end brotli incremental yield through CE | Feed byte-by-byte through CE's "br" coding; assert true memory-streaming behavior | 0-2 tests |

Estimated 0-2 new tests. Same as Phase 35's wrapper-pattern shipment (which shipped 0 new tests across all three tranches). Coverage from existing test suites is adequate.

### Acceptance criteria

- v0.9.0 ships with swift-brotli dep at 0.6+.
- v0.1-v0.8 APIs unchanged.
- All 91 v0.8 tests pass unchanged.
- CI green on macOS + Linux, first try.
- DocC tagline updated.
- CHANGELOG v0.9.0 entry documents the uniform-propagation + records that the codec-tier true-memory-streaming story is now COMPLETE end-to-end.
- **DocC discipline:** cross-package symbol refs use **single-backtick**. Critical for content-encoding's codec module mentions.

### Out of scope (deferred to v0.10+ or later phases)

- Window-carry ratio improvements (brotli v0.7 / deflate v0.7).
- `reset()` for decoder reuse.
- New coding additions beyond identity / br / deflate / gzip / zlib.

### Migration (v0.8 → v0.9)

**Additive only — non-breaking.** All v0.1-v0.8 APIs unchanged. The dep upgrade is purely internal; adopters require no code changes.

### Risk

**Low — pure dep bump.**

| Risk | Mitigation |
|---|---|
| brotli v0.6's state-machine path surfaces a behavioral subtlety not covered by CE's existing tests | Existing tests serve as the regression net. If failure occurs, file as a brotli v0.6 bug + fix upstream; don't paper over in wrapper. Phase 35's gzip/zlib propagation shipped clean with the same posture; same expectation applies here. |
| Cross-package DocC ref slip on codec module refs | Single-backtick discipline mandatory. Apply memory note as checklist (Gate 31 lesson). |

## Alternatives considered

See [Gate 36 retrospective](../docs/gates/2026-05-19-gate-36-retrospective.md) § Item 5 for the full candidate matrix.

## Dependencies

- swift-brotli ≥ 0.6 (true memory-streaming Decoder; shipped Phase 36A).
- No new external dependencies.

## References

- [Gate 36 retrospective](../docs/gates/2026-05-19-gate-36-retrospective.md) — anchor decision rationale.
- [RFC-0040](0040-phase-35-anchor-downstream-propagation-sweep.md) — Phase 35 (downstream propagation sweep); the wrap-pattern inheritance precedent + partial-propagation-acknowledgment sub-pattern.
- [RFC-0041](0041-phase-36-anchor-swift-brotli-v0.6-true-memory-streaming.md) — Phase 36 (brotli v0.6 true memory-streaming); the refactor being propagated here.
- [feedback_docc_cross_package](MEMORY first-line) — single-backtick for cross-package DocC refs.

## Decision

**Accepted 2026-05-19.** Phase 37 anchored on **swift-content-encoding v0.9 uniform brotli propagation**. Brainstorm + plan + execute via the bare-swift inline-execution pattern. Per-tranche calendar estimate **deferred to brainstorm reality-check** per Gate 22's canonical pattern. Phase 37 is the natural conclusion of the 7-phase codec-tier true-memory-streaming arc.

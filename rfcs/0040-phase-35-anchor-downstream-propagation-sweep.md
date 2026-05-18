# RFC-0040 — Phase 35 anchor: downstream propagation sweep (gzip v0.6 + zlib v0.6 + content-encoding v0.8)

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-18 |
| Resolution | Accepted 2026-05-18 — Phase 35 plans may begin authoring. Three existing packages: swift-gzip v0.6, swift-zlib v0.6, swift-content-encoding v0.8. |

## Summary

Anchor Phase 35 on a **3-tranche downstream propagation sweep** — bump `swift-gzip` to v0.6, `swift-zlib` to v0.6, and `swift-content-encoding` to v0.8 via dep-bump-driven inheritance from Phase 34's swift-deflate v0.6 true memory-streaming inflate refactor. Each downstream package wraps the deflate streaming decoder; bumping the deflate dep to 0.6 inherits true memory-streaming automatically. Closes the deflate-family true-memory-streaming arc started by Phase 30.

Per-tranche calendar **deferred to brainstorm reality-check** per Gate 22 canonical pattern. Wrapper-pattern calibration bucket is 30-60 min per package (Phase 26 precedent at 10 instances).

## Problem

[Gate 34 retrospective](../docs/gates/2026-05-18-gate-34-retrospective.md) item 5 requires "Phase 35 anchor decision recorded as an RFC" before Phase 35 plans begin. The retrospective surveyed ten candidate waves; this RFC formalizes the choice.

**Concrete gap:** Phase 34 shipped swift-deflate v0.6 with true memory-streaming inflate. swift-gzip v0.5, swift-zlib v0.5, and swift-content-encoding v0.7 still depend on swift-deflate v0.5 (buffering-wrap inflate). Their `Streaming.Decoder` types wrap `Deflate.Streaming.Decoder`; bumping the deflate dep to 0.6 inherits true memory-streaming automatically — the wrapper-pattern does the work.

**Why now (Phase 35):**
1. Phase 34's deflate v0.6 refactor is the foundation; Phase 35 propagates it to all downstream consumers.
2. Wrapper-pattern inheritance is mechanical — Phase 31's precedent (gzip + zlib v0.5 inheriting deflate v0.5's buffering-wrap) confirms this works.
3. HTTP middleware adopters consuming through content-encoding v0.7 get true memory-streaming decode end-to-end for free.
4. Closes the deflate-family true-memory-streaming arc (Phase 30 → 31 → 34 → 35). Coherent multi-phase project conclusion.
5. Validates Phase 34's refactor under composition. If gzip/zlib v0.6 inheritance is clean (existing test suites pass unchanged), the refactor is confirmed correct under wrapper-pattern use.
6. Lowest risk on the candidate queue — pure dep-bumps, no logic changes, regression on existing tests sufficient.

**Why not the alternatives (full survey in Gate 34 retro § Item 5):**
- **swift-brotli v0.6 true memory-streaming inflate** — higher risk than deflate; defer to Phase 36+ once Phase 35 proves the propagation pattern.
- **swift-oauth2-client v0.4** — v0.3 shipped Phase 29; no concrete adopter demand surfaced.
- **brotli / deflate v0.6 window carry** — ratio improvement; not blocking.
- **B3 + Jaeger, idna v0.3** — long-running correct-deferrals; no demand.
- **RS-family JWT** — STILL BLOCKED on swift-crypto `_RSA` SPI.
- **Crypto-adjacent** — 29th rejection.
- **Package rename** — premature.

## Proposal

### Anchor: 3-tranche downstream propagation sweep

| Tranche | Package | Version bump | Scope |
|---|---|---|---|
| 35A | swift-gzip | 0.5.0 → 0.6.0 | Package.swift swift-deflate dep `from: "0.5.0"` → `from: "0.6.0"`. Run existing tests; verify pass unchanged. CHANGELOG v0.6.0 note: inheritance via wrap. |
| 35B | swift-zlib | 0.5.0 → 0.6.0 | Same pattern as 35A. |
| 35C | swift-content-encoding | 0.7.0 → 0.8.0 | Package.swift codec deps: deflate/gzip/zlib bumped to 0.6.0; brotli stays at 0.5.0 (Phase 36+ candidate). Partial-propagation note in CHANGELOG. Run existing tests. |

### Brainstorm decision: Shape A (3 tranches) vs Shape B (2 tranches)

Per the 4-instance brainstorm-empowered-by-RFC scope simplification pattern, brainstorm chooses between:

- **Shape A (recommended): 3 tranches.** 35A swift-gzip + 35B swift-zlib + 35C swift-content-encoding. More discoverable in git history; aligns with Phase 31 + Phase 27 multi-tranche precedents.
- **Shape B: 2 tranches.** 35A swift-gzip + swift-zlib coordinated (same-day); 35B swift-content-encoding. More efficient; matches Phase 31's 2-tranche shape.

Both defensible. Brainstorm reality-checks whether either gzip or zlib has any test that would surface behavioral subtleties; if not, Shape B is the efficient default.

### Brainstorm decision: ship swift-content-encoding v0.8 in Phase 35 or defer to Phase 36+?

Two options:
- **Option 1 (recommended pending reality-check): ship v0.8 in Phase 35 with partial propagation.** content-encoding v0.8 bumps deflate/gzip/zlib to 0.6 but leaves brotli at 0.5. The brotli codec wrap remains buffering-wrap until Phase 36+ ships brotli v0.6. Document honestly in v0.8 CHANGELOG. **New sub-pattern: partial-propagation-acknowledgment** in version bumps where some downstream deps have moved and others haven't.
- **Option 2: defer v0.8 to Phase 36+.** Ship Phase 35 as 35A + 35B (gzip + zlib only). Content-encoding waits until brotli v0.6 also lands so v0.8 propagates uniformly to all four codecs.

Option 1 maximizes value delivery and codifies the partial-propagation pattern (which will arise again in future phases). Option 2 keeps content-encoding propagation uniform but delays the benefit. Brainstorm reality-checks the design intent.

### Test surface

Per-tranche: **no new tests required.** Existing v0.5 test suites must continue to pass unchanged. This is the canonical wrapper-pattern test posture (Phase 31 precedent).

Optional: 1-2 tests per package that exercise the v0.6 incremental yield through the wrap (e.g., feed bytes byte-by-byte; verify wrap behavior matches v0.6 deflate's incremental yield). Worth doing in 35A or 35B if reality-check identifies a clean test seam.

### Acceptance criteria

- v0.6.0 (gzip), v0.6.0 (zlib), and v0.8.0 (content-encoding) all ship with deflate dep at 0.6+.
- v0.1-v0.5 / v0.1-v0.7 APIs unchanged on each package.
- All existing tests pass unchanged.
- CI green on macOS + Linux, first try.
- DocC tagline updated to mention v0.6 / v0.8 inheritance.
- CHANGELOG v0.6.0 / v0.8.0 entries document the propagation + partial-propagation acknowledgment (content-encoding v0.8 case).
- **DocC discipline:** cross-package symbol refs use **single-backtick**. Critical for content-encoding v0.8 which references codec types.

### Out of scope (deferred to v0.7+ or later phases)

- swift-brotli v0.6 true memory-streaming inflate (Phase 36+).
- swift-content-encoding v0.9 with full brotli propagation (Phase 36+ after brotli v0.6).
- Window-carry ratio improvements (Phase 36+ or later).
- `reset()` for decoder reuse.

### Migration (per package)

**Additive only — non-breaking on each package.** All prior APIs unchanged.

### Risk

**Low per brainstorm-locked shape:**
- **Shape A (3 tranches)**: Low risk on each tranche. Pure dep-bumps. ~30-60 min × 3 = ~1.5-3 hr total.
- **Shape B (2 tranches)**: Same risk. ~1-2 hr (gzip+zlib coordinated) + ~30-60 min (content-encoding) = ~1.5-3 hr total.

| Risk | Mitigation |
|---|---|
| gzip or zlib test surfaces a behavioral subtlety not covered by deflate's v0.6 tests | Existing tests serve as the regression net. If failure occurs, file as a deflate v0.6 bug + fix upstream; don't paper over in wrapper. |
| content-encoding partial-propagation creates inconsistent adopter experience (deflate/gzip/zlib stream truly; brotli still buffers) | Document explicitly in v0.8 CHANGELOG. Codify the partial-propagation-acknowledgment sub-pattern. Plan brotli v0.6 + content-encoding v0.9 follow-up. |
| Cross-package DocC ref slip on codec module refs | Single-backtick discipline mandatory. Apply memory note as checklist (Gate 31 lesson). |

## Alternatives considered

See [Gate 34 retrospective](../docs/gates/2026-05-18-gate-34-retrospective.md) § Item 5 for the full candidate matrix.

## Dependencies

- swift-deflate ≥ 0.6 (true memory-streaming Inflater; shipped Phase 34A).
- No new external dependencies.

## References

- [Gate 34 retrospective](../docs/gates/2026-05-18-gate-34-retrospective.md) — anchor decision rationale.
- [RFC-0036](0036-phase-31-anchor-gzip-zlib-v0.5-streaming-inflate.md) — Phase 31 (gzip + zlib v0.5 wrap-pattern); the inheritance-via-dep-bump precedent.
- [RFC-0038](0038-phase-33-anchor-swift-content-encoding-v0.7-streaming-decode.md) — Phase 33 (content-encoding v0.7); previous content-encoding bump pattern.
- [RFC-0039](0039-phase-34-anchor-swift-deflate-v0.6-true-memory-streaming.md) — Phase 34 (deflate v0.6 true memory-streaming); the refactor being propagated here.
- [feedback_docc_cross_package](MEMORY first-line) — single-backtick for cross-package DocC refs.

## Decision

**Accepted 2026-05-18.** Phase 35 anchored on **downstream propagation sweep** (swift-gzip v0.6 + swift-zlib v0.6 + swift-content-encoding v0.8). Brainstorm + plan + execute via the bare-swift inline-execution pattern. Per-tranche calendar estimate **deferred to brainstorm reality-check** per Gate 22's canonical pattern. Shape A vs Shape B + content-encoding ship-or-defer decided by brainstorm.

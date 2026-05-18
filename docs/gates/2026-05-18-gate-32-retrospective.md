# Gate 32 Retrospective: Phase 32 → Phase 33

**Date:** 2026-05-18
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 32 (the swift-brotli v0.5 streaming-inflate tranche anchored by [RFC-0037](../../rfcs/0037-phase-32-anchor-swift-brotli-v0.5-streaming-inflate.md)) against the Gate 1–31 criteria template and recommends whether Phase 33 should start. Phase 32 was a **single-tranche existing-package minor-bump phase** (32A swift-brotli v0.5). It completes the codec-tier streaming-decode sweep started in Phase 30.

**Notable:** Phase 32A landed first-try-clean on CI, restarting the streak at 1 after the Phase 31A break. In-package-only DocC symbol refs avoided the cross-package trap that broke Phase 31A.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All anchored Tranche 32A deliverables shipped, CI green, DocC live, index bumped | ✓ PASS |
| 2 | RFC-0037 commitments fulfilled | ✓ PASS *(zero scope reshape; 16th consecutive clean RFC since Phase 17)* |
| 3 | ≥1 substantive RFC accepted *or documented absence of policy gaps* | ✓ PASS (no new RFCs needed; codec-tier streaming-decode sweep closes per established patterns) |
| 4 | RFC-0001 conventions stress-test against Phase 32 deliverables | ✓ DONE BELOW |
| 5 | Phase 33 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0038 below) |

**Overall:** Items 1–4 pass. Item 5 is open. Phase 33 plans should be drafted but not executed until item 5 closes via RFC-0038.

Phase 32 calendar time was ~30-40 min wall-clock (within RFC-0037's 30-60 min Shape A bracket; below the 1-2 hour wrapper-pattern bracket).

---

## Item 1 — Phase 32 deliverables shipped ✓ PASS

| Tranche | Package | Tag | New tests | DocC | Index | CI |
|---|---|---|---|---|---|---|
| 32A | swift-brotli | **v0.5.0** | 17 (141 total) | ✓ | ✓ (0.4.0→0.5.0) | green (first-try) |

Single-tranche. **Codec-tier streaming-decode sweep COMPLETE** (deflate + gzip + zlib + brotli all stream-decode-capable).

**Phase 32 final tally:**
- 32A `Brotli.Streaming.Decoder` (Sendable value-type; init/update/finish; buffering-wrap internals).
- New error case `BrotliError.decoderFinished`.
- ~50 source LOC + ~200 test LOC (single tranche).
- ~30-40 min wall-clock — mid-Shape-A bracket per RFC-0037.
- **First-try-clean CI streak restarted at 1** (after Phase 31A break at 19).
- **16-consecutive zero-scope-reshape RFC streak** preserved.
- v0.1-v0.4 byte-for-byte preserved; new error case additive.

**Streaming-decode sweep status (post-Phase 32A):**
- ✓ Phase 30A: deflate v0.5 (foundation; buffering wrap)
- ✓ Phase 31A: gzip v0.5 (wrapper over deflate)
- ✓ Phase 31B: zlib v0.5 (wrapper over deflate)
- ✓ **Phase 32A: brotli v0.5 (independent buffering wrap)**
- 🟡 Phase 33+: content-encoding v0.7 streaming-decode wiring (**now unblocked**)
- 🟡 v0.6+: deflate / brotli true memory-streaming inflate (state-machine refactors; demand-driven)

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists **49 packages** — no count change.

---

## Item 2 — RFC-0037 commitments ✓ PASS (zero scope reshape)

[RFC-0037](../../rfcs/0037-phase-32-anchor-swift-brotli-v0.5-streaming-inflate.md) committed to:

| RFC-0037 commitment | Outcome |
|---|---|
| One existing package: swift-brotli v0.4 → v0.5 | ✓ shipped. |
| `Brotli.Streaming.Decoder` mirroring `Brotli.Streaming.Encoder` shape (init/update/finish) | ✓ shipped. |
| Buffering-wrap (Shape A) or state-machine refactor (Shape B) — brainstorm decides | ✓ Shape A chosen per Phase 30 precedent + reality-check on Decoder.swift complexity. |
| `BrotliError.decoderFinished` error case | ✓ shipped. |
| Calendar estimate **deferred to brainstorm reality-check** | ✓ honored. Actual 30-40 min within Shape A's 30-60 min bracket. |
| ~12-15 tests (RFC range) | ✓ 17 new tests (above mid-bracket). |
| Non-breaking on v0.1-v0.4 | ✓ confirmed. |
| Cross-package DocC discipline re-emphasized | ✓ honored — Phase 32A used **only** in-package symbol refs; no cross-package refs needed since `Brotli.Streaming.Decoder` docstring references only `Brotli.decode`, `Brotli.Streaming.Encoder`, and `BrotliError.decoderFinished` (all same package). |

**Zero scope reshape this phase.** **16th consecutive clean-from-RFC phase** (Phases 17-32).

**Estimate-vs-actual:** RFC-0037 said "~30-60 min for Shape A"; actual ~30-40 min (lower-mid bracket). 11th successful reality-check application.

---

## Item 3 — Revised criterion ✓ PASS

**Pre-Phase 32 (Gate 31 closeout):** RFC-0037 accepted 2026-05-17.

**During Phase 32 execution:** zero RFCs accepted.

Phase 32 reinforced existing patterns and surfaced no new gaps:

- **Honest-scope-under-limitation at 5 instances** (Phase 25 → 28 → 30 → 31 → 32). Procedural law fully canonical. Brotli v0.5 inherits the same buffering-wrap-with-deferred-state-machine-refactor pattern from deflate v0.5 + gzip v0.5 + zlib v0.5.
- **Brainstorm-empowered-by-RFC scope simplification at 2 instances** (Phase 30 + 32; both chose Shape A buffering-wrap over Shape B state-machine refactor based on reality-checked complexity of the foundation decoder). Becoming a canonical sub-pattern.
- **Reality-check-before-RFC-estimate at 11/11 successful applications.** Procedural law.
- **DocC discipline maintained.** Phase 32A used in-package refs only; the long-standing first-line memory note was applied as a checklist per Gate 31's lesson. Vigilance reset held.
- **Streaming-symmetric API surface ahead of true streaming.** Codified observation: v0.5 ships the API shape that downstream consumers adopt today; the future v0.6+ state-machine refactor will not change the shape. Adopters pay no migration cost. This is a **new sub-pattern** worth noting — explicit decoupling of "API surface" from "internal implementation depth" in versioning strategy.

None warrant a new cross-package RFC. The honest-scope-under-limitation pattern is fully codified; the streaming-symmetric-API-ahead-of-true-streaming observation is a refinement of it.

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

| Convention | swift-brotli v0.5 |
|---|---|
| Module name PascalCase | ✓ Brotli |
| Apache-2.0 + LLVM exception SPDX | ✓ |
| Sendable-clean | ✓ |
| Single public error enum | ✓ BrotliError (15 cases: 14 prior + 1 new `decoderFinished`) |
| Public APIs Foundation-free | ✓ |
| README + DocC updated | ✗ (README not updated for v0.5; only CHANGELOG + DocC.docc covered) — minor docs gap |
| CHANGELOG with v0.5.0 entry | ✓ (includes honest-scope note) |
| CI green | ✓ first-try |
| DocC bundle | ✓ first-try (`swift package generate-documentation --warnings-as-errors` clean local) |
| Sanitizers ON | ✓ |
| Backwards-compat preservation | ✓ v0.1-v0.4 byte-for-byte; new error case additive |

### Deviations and findings

**1. README update not shipped.** swift-brotli v0.5 got CHANGELOG entry + DocC update but no README example for streaming decode. **Tradeoff:** matches the Gate 27 + Gate 31 "README-deferral-during-orchestration-sweeps" pattern at **3rd instance** — streaming-decode is primarily used through content-encoding (Phase 33+); adopters don't typically need a direct brotli streaming-decode example in the brotli README. **Memory:** acceptable trade-off for orchestration-primitive sweeps; revisit when content-encoding v0.7 ships (Phase 33+).

**2. Honest-scope-under-limitation pattern at 5 instances.** Procedural law fully canonical. Brotli v0.5 inherits the same deferral arc as deflate v0.5 + gzip v0.5 + zlib v0.5.

**3. DocC discipline held.** Phase 32A used in-package refs only — `Brotli.decode`, `Brotli.Streaming.Encoder`, `BrotliError.decoderFinished`. No cross-package double-backtick risk. The Phase 31A vigilance-reset lesson was applied as a checklist before authoring docstrings.

**4. Streaming-symmetric API surface ahead of true streaming.** Codified as a refinement of the honest-scope-under-limitation pattern. v0.5 ships the API shape (init/update/finish) that downstream consumers can adopt today; the v0.6+ state-machine internals refactor will not change the shape. Adopters pay no migration cost when memory-streaming lands.

### Verdict

RFC-0001 held cleanly. One minor docs gap (README deferred — acceptable trade-off; 3rd instance of the established pattern). Gate 32 confirms: in-package-symbol-ref discipline plus first-line-memory-note-as-checklist discipline together kept the CI streak restart clean.

---

## Item 5 — Phase 33 anchor decision ✗ NOT YET (recommendation)

Phase 33 anchor candidates surveyed:

| Wave | Packages | Risk | Adoption fit | Deferral count |
|---|---|---|---|---|
| **swift-content-encoding v0.7 streaming-decode wiring** | content-encoding v0.7 | Low-medium | **Highest** — all 4 codec streaming-decoders shipped (Phases 30-32). HTTP body streaming-decode story closes. | 0 (was blocked on Phase 32; now unblocked) |
| swift-deflate v0.6 true memory-streaming inflate | deflate v0.6 | High (state-machine refactor) | Medium — v0.5 just shipped; adopter demand for memory-streaming unclear; would unlock gzip/zlib v0.6 inheritance. | 1 (just deferred from v0.5) |
| swift-brotli v0.6 true memory-streaming inflate | brotli v0.6 | High (state-machine refactor; more complex than deflate) | Medium — v0.5 just shipped; demand-driven. | 0 (just deferred from v0.5) |
| swift-brotli v0.6 + swift-deflate v0.6 window carry | 2 packages | Medium | Low-medium — ratio improvement. | 8 each (Phases 25-32) |
| swift-oauth2-client v0.4 | oauth2-client v0.4 | Medium | Low — v0.3 shipped Phase 29; some settle time but no concrete demand. | 1 (Phase 32) |
| swift-distributed-tracing-bridge v0.3 (B3 + Jaeger) | bridge v0.3 | Low-medium | Low — 13 deferrals; no concrete demand. | 13 |
| swift-idna v0.3 (Bidi + ContextJ + ContextO) | swift-idna v0.3 | Medium | Low — rarely-hit Unicode rules. | 17 |
| RS-family JWT algorithms | swift-jwt-verify v0.4 | High | Low — **STILL BLOCKED** on swift-crypto `_RSA` SPI. | n/a (blocked) |
| Crypto-adjacent | various | Medium-low | Low — 28x rejected. | 28 (rejected) |
| Package rename `swift-jwt-verify` → `swift-jwt` | rename project | Low | Low-medium | n/a |

### Recommendation: **Anchor on swift-content-encoding v0.7 (streaming-decode wiring)**

Reasons:

1. **Now unblocked.** All 4 codec streaming-decoders shipped (Phases 30-32). This is the natural sequel to the codec-tier sweep and the third instance of "codec-tier ships → content-encoding wires it" (after v0.5 single-coding streaming-encode and v0.6 multi-coding streaming-encode).
2. **Closes HTTP body streaming story end-to-end.** Encode side complete via Phase 28; decode side currently has only one-shot. Phase 33 closes symmetry.
3. **Highest adoption fit.** HTTP middleware adopters need streaming-decode on the response path (incoming `Content-Encoding`) just as much as streaming-encode on the request path. Phase 33 delivers that.
4. **Low-medium risk.** The architectural pattern is already established (Phase 25's `InnerEncoder` dispatch enum for single-coding streaming-encode; Phase 28's cascaded-pipeline for multi-coding streaming-encode). Phase 33 mirrors these on the decode side.
5. **Modest scope.** Likely 1-2 tranches (single-coding streaming-decode + multi-coding cascaded streaming-decode). Reality-check defers calendar estimate to brainstorm.
6. **Validates the buffering-wrap honest-scope choice.** Confirms that the v0.5-v0.5-v0.5-v0.5 buffering-wrap quartet composes correctly through content-encoding's higher-level streaming machinery. If it does, the v0.6+ true-memory-streaming refactors become genuinely demand-driven (not capability-blocking).
7. **Audience continuity.** Same HTTP-codec adopters from Phases 22-32.
8. **Non-breaking additive minor bump.** All v0.1-v0.6 APIs unchanged.

### Phase 33 tranche sketch (subject to formal RFC + brainstorm reality-check)

**Shape A (recommended pending reality-check): single tranche — single-coding streaming-decode.**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 33A | swift-content-encoding v0.7 | Single-coding streaming-decode via `InnerDecoder` dispatch enum over the 4 streaming decoders + ~12-15 tests | ~1-2 hours (pending reality-check) |

**Shape B (alternate): two tranches — single + multi-coding.**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 33A | swift-content-encoding v0.7 | Single-coding streaming-decode | ~1-2 hours |
| 33B | swift-content-encoding v0.7 | Multi-coding streaming-decode via cascaded pipeline (mirror of Phase 28) | ~1-2 hours |

Decision deferred to brainstorm. Shape A is conservative (closes the most-common 95%+ case first); Shape B closes the symmetric end-to-end story in one phase. Phase 28 precedent suggests Shape B is feasible same-phase if reality-check confirms the cascaded-pipeline pattern mirrors cleanly to decode.

### Why other waves were rejected for Phase 33

- **swift-deflate v0.6 + swift-brotli v0.6 true memory-streaming** — v0.5 just shipped; demand-driven; content-encoding v0.7 will reveal whether v0.5's buffering-wrap is actually a bottleneck.
- **brotli / deflate v0.6 window carry** — ratio improvement; not blocking.
- **swift-oauth2-client v0.4** — v0.3 shipped Phase 29; no concrete adopter demand surfaced.
- **B3 + Jaeger, idna v0.3** — long-running correct-deferrals; no demand.
- **RS-family JWT** — blocked on swift-crypto `_RSA` SPI.
- **Crypto-adjacent** — 28th rejection.
- **Package rename** — premature.

**Action:** author the anchor decision as **RFC-0038** (Phase 33 anchor: swift-content-encoding v0.7 streaming-decode wiring) and accept it before Phase 33 plans start.

---

## Open work to clear Gate 32

1. **Write RFC-0038** (Phase 33 anchor: swift-content-encoding v0.7 streaming-decode wiring). ~1-hour task.
2. **Begin Phase 33 brainstorm + reality-check** after RFC-0038 accepts. Brainstorm should:
   - Survey content-encoding v0.6's `InnerEncoder` dispatch enum + cascaded pipeline; decide whether the decode-side mirror is one tranche or two.
   - Decide Shape A (single-coding only, 1 tranche) vs Shape B (single + multi-coding, 2 tranches). Either is defensible.
   - **Maintain DocC discipline:** cross-package refs use single-backtick. Codec module refs from content-encoding docstrings will need this care.

---

## What this retrospective changes for Phase 33

- The plan-then-execute-inline workflow stays.
- **CI first-try-clean streak at 1** (Phase 32A; restart after Phase 31A break).
- **Honest-scope-under-limitation pattern at 5 instances** — procedural law fully canonical. Will be exercised again if Phase 33 surfaces any limitation; expected no.
- **Brainstorm-empowered-by-RFC scope simplification at 2 instances** — becoming a canonical sub-pattern.
- **Streaming-symmetric-API-surface-ahead-of-true-streaming** — new sub-pattern of honest-scope-under-limitation; codified for future versioning decisions.
- **Reality-check-before-RFC-estimate at 11/11.**
- **README-deferral-during-orchestration-sweeps pattern at 3rd instance** — procedural acceptance.
- **Codec-tier streaming-decode sweep COMPLETE** — Phase 33 begins the content-encoding integration tier.

---

## Decision

Gate 32 is **PASSED on items 1, 2, 3, 4** and **NOT YET PASSED on item 5 (open)**. Phase 33 plans should be drafted but not executed until item 5 closes via RFC-0038.

Phase 32 closed the codec-tier streaming-decode sweep (brotli v0.5 streaming inflate; final codec). 16-consecutive zero-scope-reshape RFC streak preserved. First-try-clean CI streak restarted at 1 after Phase 31A's break. In-package-symbol-ref discipline plus first-line-memory-note-as-checklist discipline together kept the restart clean.

The next step is **RFC-0038 anchoring Phase 33 on swift-content-encoding v0.7 streaming-decode wiring** — completes HTTP body streaming end-to-end (codec-tier + content-encoding tier both stream-decode-capable). swift-deflate v0.6 + swift-brotli v0.6 true memory-streaming, swift-oauth2-client v0.4, brotli + deflate v0.6 window carry, B3 + Jaeger, idna v0.3, RS-family JWT (when swift-crypto stabilizes `_RSA`), and package rename remain on the queue for Phase 34+.

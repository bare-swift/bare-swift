# Gate 35 Retrospective: Phase 35 → Phase 36

**Date:** 2026-05-18
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 35 (the 3-tranche downstream propagation sweep anchored by [RFC-0040](../../rfcs/0040-phase-35-anchor-downstream-propagation-sweep.md)) against the Gate 1–34 criteria template and recommends whether Phase 36 should start. Phase 35 was a **3-tranche existing-packages dep-bump sweep** (35A swift-gzip v0.6, 35B swift-zlib v0.6, 35C swift-content-encoding v0.8 with partial brotli propagation).

**Notable:** Phase 35 closes the **deflate-family true-memory-streaming arc** (Phase 30 → 31 → 34 → 35). Three RESOLUTIONS of the honest-scope-under-limitation pattern logged in 24 hours.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All anchored Tranche 35A + 35B + 35C deliverables shipped, CI green, DocC live, index bumped | ✓ PASS |
| 2 | RFC-0040 commitments fulfilled | ✓ PASS *(zero scope reshape; 19th consecutive clean RFC since Phase 17)* |
| 3 | ≥1 substantive RFC accepted *or documented absence of policy gaps* | ✓ PASS (no new RFCs needed; one new sub-pattern codified: partial-propagation-acknowledgment) |
| 4 | RFC-0001 conventions stress-test against Phase 35 deliverables | ✓ DONE BELOW |
| 5 | Phase 36 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0041 below) |

**Overall:** Items 1–4 pass. Item 5 is open. Phase 36 plans should be drafted but not executed until item 5 closes via RFC-0041.

Phase 35 calendar time was ~45-50 min wall-clock total across all three tranches (extremely efficient — well under RFC-0040's 1.5-3 hr Shape A bracket).

---

## Item 1 — Phase 35 deliverables shipped ✓ PASS

| Tranche | Package | Tag | New tests | DocC | Index | CI |
|---|---|---|---|---|---|---|
| 35A | swift-gzip | **v0.6.0** | 0 (63 unchanged) | ✓ | ✓ (0.5.0→0.6.0) | green (first-try) |
| 35B | swift-zlib | **v0.6.0** | 0 (57 unchanged) | ✓ | ✓ (0.5.0→0.6.0) | green (first-try) |
| 35C | swift-content-encoding | **v0.8.0** | 0 (91 unchanged) | ✓ | ✓ (0.7.0→0.8.0) | green (first-try) |

3 tranches; all shipped same-day. **Deflate-family true-memory-streaming arc CLOSED.**

**Phase 35 final tally:**
- 35A `swift-gzip v0.6.0`: swift-deflate dep 0.5 → 0.6. Public API surface byte-for-byte preserved.
- 35B `swift-zlib v0.6.0`: swift-deflate dep 0.5 → 0.6. Public API surface byte-for-byte preserved.
- 35C `swift-content-encoding v0.8.0`: deflate/gzip/zlib deps 0.5 → 0.6; brotli intentionally pinned at 0.5 (Phase 36+ candidate). Public API surface byte-for-byte preserved.
- Zero new tests across all three; existing 211 v0.5/v0.7 tests pass without modification.
- ~45-50 min wall-clock total — well under RFC-0040's 1.5-3 hr Shape A bracket.
- **First-try-clean CI streak NOW AT 6** (Phase 32A + 33A + 34A + 35A + 35B + 35C after Phase 31A break).
- **19-consecutive zero-scope-reshape RFC streak** preserved.
- v0.1-v0.5 / v0.1-v0.7 byte-for-byte preserved across all three packages.

**Three RESOLUTIONS of honest-scope-under-limitation logged in Phase 35:**
- 35A: gzip v0.5's buffering-wrap-via-Deflate.Streaming.Decoder resolved by inheritance.
- 35B: zlib v0.5's buffering-wrap-via-Deflate.Streaming.Decoder resolved by inheritance.
- 35C (partial): content-encoding v0.7's buffering-wrap for deflate/gzip/zlib paths resolved; brotli path remains buffering-wrap until brotli v0.6 (Phase 36+).

**Streaming story status (post-Phase 35) — DEFLATE-FAMILY COMPLETE:**
- ✓ Codec-tier streaming-encode (v0.3+)
- ✓ Codec-tier drain() (v0.4)
- ✓ Codec-tier streaming-decode API surface (v0.5)
- ✓ **Codec-tier true memory-streaming decode for deflate/gzip/zlib (v0.6)** — Phase 34A + 35A + 35B
- ✓ Content-encoding streaming-encode (v0.5, v0.6)
- ✓ Content-encoding streaming-decode (v0.7, v0.8)
- ✓ **Content-encoding true memory-streaming for deflate/gzip/zlib chains (v0.8)** — Phase 35C
- 🟡 Codec-tier true memory-streaming for brotli (Phase 36+ independent refactor)
- 🟡 Content-encoding uniform propagation v0.9 (post-brotli-v0.6)

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists **49 packages** — no count change.

---

## Item 2 — RFC-0040 commitments ✓ PASS (zero scope reshape)

[RFC-0040](../../rfcs/0040-phase-35-anchor-downstream-propagation-sweep.md) committed to:

| RFC-0040 commitment | Outcome |
|---|---|
| Three existing packages: gzip 0.5→0.6, zlib 0.5→0.6, content-encoding 0.7→0.8 | ✓ all shipped. |
| Shape A (3 tranches) or Shape B (2 tranches; gzip + zlib coordinated) — brainstorm decides | ✓ **Shape A chosen** for clean git history; aligns with multi-tranche-phase precedent (6 instances now). |
| Content-encoding v0.8 ship-or-defer with partial brotli propagation | ✓ **Ship in Phase 35** chosen; partial-propagation-acknowledgment sub-pattern codified. |
| Per-tranche calendar **deferred to brainstorm reality-check** (30-60 min/tranche) | ✓ honored. Actual ~10-20 min/tranche — well under bracket. 14th successful reality-check application. |
| All existing tests pass unchanged on each tranche | ✓ 211 total tests (63 + 57 + 91) pass without modification. |
| Cross-package DocC discipline | ✓ honored — all DocC tagline updates use single-backtick on cross-package refs (no double-backtick slips). |
| Non-breaking on each package | ✓ confirmed (v0.5 → v0.6 minor bumps on gzip/zlib; v0.7 → v0.8 minor bump on content-encoding). |

**Zero scope reshape this phase.** **19th consecutive clean-from-RFC phase** (Phases 17-35).

**Estimate-vs-actual:** RFC-0040 said "~1.5-3 hr total"; actual ~45-50 min (well under bracket; under low-end estimate by ~50%). 14th successful reality-check application. **Wrapper-pattern calibration bucket (30-60 min/package; 11 prior instances) underperformed at ~10-20 min/tranche** — suggests pure dep-bumps without any logic changes form a sub-category that calibrates closer to 10-20 min. Worth tracking whether this is consistent.

---

## Item 3 — Revised criterion ✓ PASS

**Pre-Phase 35 (Gate 34 closeout):** RFC-0040 accepted 2026-05-18.

**During Phase 35 execution:** zero RFCs accepted.

Phase 35 codified one new sub-pattern and reinforced existing patterns:

- **NEW sub-pattern: partial-propagation-acknowledgment.** When propagating an upstream refactor through downstream wrappers, if not all upstream packages have been refactored, downstream packages SHOULD pin the un-refactored upstream and document the inconsistency honestly in CHANGELOG. Codified for the swift-content-encoding v0.8 case: deflate/gzip/zlib at 0.6, brotli at 0.5. Adopters using multi-coding chains entirely within deflate-family get true-memory-streaming; chains including brotli still buffer through the brotli stage. **Decision principle:** ship value incrementally with explicit scope acknowledgment, rather than defer the downstream release until uniform propagation is possible. Most adopters benefit from partial propagation immediately.
- **Honest-scope-under-limitation pattern: 6 instances, 3 RESOLUTIONS in two consecutive phases (Phase 34 + 35).** Pattern lifecycle now well-trodden — ship streaming-symmetric API surface in vN under honest-scope-limitation deferral → resolve in vN+1 via refactor or inheritance → adopters pay zero migration cost. The 3 resolutions in 24 hours validates the design intent thoroughly.
- **Wrapper-pattern inheritance under refactor.** Phase 34 proved a state-machine refactor inherits cleanly via dep-bump. Phase 35 35A + 35B execute the inheritance at scale on two wrapper packages simultaneously — no behavioral subtleties surfaced. Confirms Phase 31 precedent in the inverse direction.
- **Multi-tranche same-day shipping at 3-tranche size.** Demonstrates 3 mechanical tranches can ship in a single session with full ceremony (CI, tag, Release + Publish-docs, umbrella bump). ~45-50 min total wall-clock including all workflow watches. Sets calibration baseline for future 3-tranche propagation sweeps.
- **Reality-check-before-RFC-estimate at 14/14 successful applications.** Procedural law fully canonical.
- **Coordinated-version-bump-across-deps at 6 instances** (Phase 24, 27, 28, 31, 33, 35). Phase 35 was the largest coordinated bump (3 packages same-day) — sets ceiling-of-coordination calibration.

None warrant a new cross-package RFC. The partial-propagation-acknowledgment pattern is documented in this retrospective, the Phase 35 memory closure, and the swift-content-encoding v0.8 CHANGELOG entry — sufficient for future reference.

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

| Convention | swift-gzip v0.6 | swift-zlib v0.6 | swift-content-encoding v0.8 |
|---|---|---|---|
| Module name PascalCase | ✓ Gzip | ✓ Zlib | ✓ ContentEncoding |
| Apache-2.0 + LLVM exception SPDX | ✓ | ✓ | ✓ |
| Sendable-clean | ✓ | ✓ | ✓ |
| Single public error enum | ✓ GzipError (cases unchanged) | ✓ ZlibError (cases unchanged) | ✓ ContentEncodingError (cases unchanged) |
| Public APIs Foundation-free | ✓ | ✓ | ✓ |
| README + DocC updated | Partial (DocC tagline only) | Partial (DocC tagline only) | Partial (DocC tagline only) |
| CHANGELOG with v0.6.0/v0.8.0 entry | ✓ inheritance note + migration | ✓ inheritance note + migration | ✓ partial-propagation note + memory implication |
| CI green | ✓ first-try | ✓ first-try | ✓ first-try |
| DocC bundle | ✓ first-try | ✓ first-try | ✓ first-try |
| Sanitizers ON | ✓ | ✓ | ✓ |
| Backwards-compat preservation | ✓ all 63 tests pass unchanged | ✓ all 57 tests pass unchanged | ✓ all 91 tests pass unchanged |

### Deviations and findings

**1. README not updated on any of the three packages.** **README-deferral-during-orchestration-sweeps pattern at 6 instances** (Phase 27, 31, 32, 33, 34, 35). Fully canonical procedural acceptance. CHANGELOG entries are the canonical adopter-facing reference for these dep-bump releases.

**2. Three RESOLUTIONS in one phase.** This is the first multi-resolution phase. Validates that wrapper-pattern inheritance can resolve multiple honest-scope-under-limitation instances simultaneously via a single coordinated dep-bump sweep. Worth tracking as a "resolution density" metric for future phases.

**3. Partial-propagation choice was non-obvious.** Two options were on the table: ship v0.8 with partial brotli propagation, OR defer v0.8 until brotli v0.6 ships. Shipping partial maximizes value delivery for the 95%+ case (gzip/deflate are the dominant Content-Encoding values in HTTP); deferring would have given uniform propagation but blocked all adopters on brotli v0.6's higher-risk state-machine refactor. **Memory:** when a downstream propagation can land partially with honest acknowledgment, prefer partial ship over uniform deferral — codified in the partial-propagation-acknowledgment sub-pattern.

**4. Wrapper-pattern calibration sub-category.** Pure dep-bumps without logic changes calibrate at ~10-20 min/tranche, well under the 30-60 min wrapper-pattern bracket. Worth tracking whether this is a stable sub-bucket. **Memory:** for future pure-propagation phases, the bracket can be tightened.

### Verdict

RFC-0001 held cleanly across all three packages. One minor docs gap (README deferred — 6th instance of established pattern). The partial-propagation choice was deliberate and well-documented. Wrapper-pattern calibration may need a sub-bucket for pure dep-bumps.

---

## Item 5 — Phase 36 anchor decision ✗ NOT YET (recommendation)

Phase 36 anchor candidates surveyed:

| Wave | Packages | Risk | Adoption fit | Deferral count |
|---|---|---|---|---|
| **swift-brotli v0.6 true memory-streaming inflate** | brotli v0.6 | High (state-machine refactor; brotli Decoder more complex than deflate Inflater — metablock structure, multiple sub-decoders, helper files: MetaBlockHeader, OutputBuffer, ContextMap, PrefixCode) | **Highest** — closes codec-tier true-memory-streaming story entirely; unlocks content-encoding v0.9 uniform propagation. Resolves the last honest-scope-under-limitation instance in the codec-tier. | 2 (Phase 34, 35) |
| swift-content-encoding v0.9 (uniform brotli propagation) | content-encoding v0.9 | Low (pure dep-bump) | Medium-high — would inherit brotli v0.6 if also shipped; otherwise blocked. | 0 |
| swift-oauth2-client v0.4 | oauth2-client v0.4 | Medium | Medium — v0.3 shipped Phase 29; reasonable settle time. | 4 (Phase 32, 33, 34, 35) |
| swift-brotli v0.6 + swift-deflate v0.6 window carry | 2 packages | Medium | Low-medium — ratio improvement only. | 11 each (Phases 25-35) |
| swift-distributed-tracing-bridge v0.3 (B3 + Jaeger) | bridge v0.3 | Low-medium | Low — 16 deferrals; no concrete demand. | 16 |
| swift-idna v0.3 (Bidi + ContextJ + ContextO) | swift-idna v0.3 | Medium | Low — rarely-hit Unicode rules. | 20 |
| RS-family JWT algorithms | swift-jwt-verify v0.4 | High | Low — **STILL BLOCKED** on swift-crypto `_RSA` SPI. | n/a (blocked) |
| Crypto-adjacent | various | Medium-low | Low — 29x rejected. | 29 (rejected) |
| Package rename `swift-jwt-verify` → `swift-jwt` | rename project | Low | Low-medium | n/a |

### Recommendation: **Anchor on swift-brotli v0.6 true memory-streaming inflate**

Reasons:

1. **Closes the codec-tier true-memory-streaming story entirely.** Phase 34 + 35 left brotli as the last codec without true memory-streaming. Phase 36 closes the codec-tier story symmetrically.
2. **Resolves the last codec-tier honest-scope-under-limitation instance.** After Phase 36, the codec-tier has no remaining buffering-wrap deferrals. Pattern fully resolved at the codec tier.
3. **Unlocks content-encoding v0.9 uniform propagation.** Phase 36+ Tranche 36B (or Phase 37) can ship CE v0.9 as a pure dep-bump replacing the v0.8 partial propagation.
4. **Validates the snapshot-and-restore pattern at scale.** Phase 34 codified the technique on deflate; Phase 36 exercises it on a more complex codec (brotli's metablock structure + multiple sub-decoders). If the pattern generalizes cleanly, it's a strong validation.
5. **Estimate per Phase 29 state-machine-work bucket: 4-6 hr** (brotli Decoder is more complex than deflate Inflater). Phase 34's deflate v0.6 came in at 2.5 hr (under the bracket); brotli is genuinely harder. Brainstorm reality-check determines whether 4-6 hr holds or needs adjustment.
6. **Audience continuity.** Same HTTP-codec adopters from Phases 22-35.

### Phase 36 tranche sketch (subject to formal RFC + brainstorm reality-check)

**Shape A (recommended pending reality-check): single tranche, brotli v0.6 state-machine refactor.**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 36A | swift-brotli v0.6 | Refactor internal Decoder.swift + supporting helpers (MetaBlockHeader, OutputBuffer, ContextMap, PrefixCode) to a state-machine yielding bytes incrementally; preserve public Brotli.Streaming.Decoder API shape; delete buffering-wrap path; tests pass | ~4-6 hours (pending reality-check) |

**Shape B (alternate): 2-tranche brotli v0.6 + content-encoding v0.9 coordinated.**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 36A | swift-brotli v0.6 | state-machine refactor | ~4-6 hours |
| 36B | swift-content-encoding v0.9 | Dep bump brotli 0.5 → 0.6; verify tests; uniform propagation note in CHANGELOG | ~15-20 min |

Shape B closes the codec-tier + content-encoding stories in one phase. Phase 35 precedent (multi-tranche same-day shipping with mechanical follow-on) supports Shape B's feasibility.

### Why other waves were rejected for Phase 36

- **swift-content-encoding v0.9 alone** — blocked on Phase 36's brotli v0.6.
- **swift-oauth2-client v0.4** — v0.3 shipped Phase 29; no concrete adopter demand surfaced.
- **brotli / deflate v0.6 window carry** — ratio improvement; not blocking.
- **B3 + Jaeger, idna v0.3** — long-running correct-deferrals; no demand.
- **RS-family JWT** — blocked on swift-crypto `_RSA` SPI.
- **Crypto-adjacent** — 29th rejection.
- **Package rename** — premature.

**Action:** author the anchor decision as **RFC-0041** (Phase 36 anchor: swift-brotli v0.6 true memory-streaming inflate) and accept it before Phase 36 plans start.

---

## Open work to clear Gate 35

1. **Write RFC-0041** (Phase 36 anchor: swift-brotli v0.6 true memory-streaming inflate). ~1-hour task.
2. **Begin Phase 36 brainstorm + reality-check** after RFC-0041 accepts. Brainstorm should:
   - Survey swift-brotli v0.5 internal `Decoder.swift` + helper files (MetaBlockHeader, OutputBuffer, ContextMap, PrefixCode) state surfaces. Brotli has more state than deflate.
   - Decide Shape A (brotli only, 1 tranche) vs Shape B (brotli + content-encoding v0.9, 2 tranches). Shape B is the natural codec-tier story closer.
   - **Apply Phase 34's four codified sub-patterns:**
     - Snapshot-and-restore for resumable state machines.
     - Real-error-capture-during-update + rethrow-at-finish.
     - Two-track coexistence (one-shot Decoder retained; new StreamingDecoder).
     - Early-out for trivially-completable state-machine cases.
   - Reality-check whether the 4-6 hr state-machine-work bracket holds for brotli's complexity. If brainstorm uncovers tractable state-machine boundaries, the bracket may overestimate; if metablock-structure handling is intricate, the bracket may underestimate.
   - Maintain DocC discipline: cross-package refs in any updated docstrings use single-backtick.

---

## What this retrospective changes for Phase 36

- The plan-then-execute-inline workflow stays.
- **CI first-try-clean streak at 6** (Phase 32A + 33A + 34A + 35A + 35B + 35C; rebuilding strongly after Phase 31A break).
- **Honest-scope-under-limitation pattern at 6 instances, 3 RESOLUTIONS logged.** Phase 36 anchors on the 4th resolution (brotli v0.5 → v0.6); after Phase 36 + Phase 36B (or Phase 37 CE v0.9), the entire 6-instance pattern is RESOLVED.
- **Brainstorm-empowered-by-RFC scope simplification at 4 instances** — strongly canonical sub-pattern.
- **Reality-check-before-RFC-estimate at 14/14.**
- **NEW sub-pattern codified:** partial-propagation-acknowledgment in version bumps.
- **README-deferral-during-orchestration-sweeps at 6 instances** — fully canonical.
- **Multi-tranche-phase precedent at 6 instances** (3-tranche shape calibrated).
- **Coordinated-version-bump-across-deps at 6 instances** (Phase 35 set ceiling at 3 packages same-day).
- **Phase 34's four sub-patterns will be exercised in Phase 36** — Phase 36 is the validation phase for those patterns at higher state-machine complexity.

---

## Decision

Gate 35 is **PASSED on items 1, 2, 3, 4** and **NOT YET PASSED on item 5 (open)**. Phase 36 plans should be drafted but not executed until item 5 closes via RFC-0041.

Phase 35 closed the deflate-family true-memory-streaming arc (Phase 30 → 31 → 34 → 35) via 3-tranche downstream propagation sweep. Three RESOLUTIONS of the honest-scope-under-limitation pattern logged in 24 hours. 19-consecutive zero-scope-reshape RFC streak preserved. First-try-clean CI streak rebuilding at 6. One new sub-pattern codified: partial-propagation-acknowledgment.

The next step is **RFC-0041 anchoring Phase 36 on swift-brotli v0.6 true memory-streaming inflate** — resolves the last codec-tier honest-scope-under-limitation instance; closes the codec-tier true-memory-streaming story entirely; unlocks swift-content-encoding v0.9 uniform propagation (Phase 36 Tranche 36B or Phase 37). swift-oauth2-client v0.4, brotli/deflate v0.6 window carry, B3 + Jaeger, idna v0.3, RS-family JWT (when swift-crypto stabilizes `_RSA`), and package rename remain on the queue for Phase 37+.

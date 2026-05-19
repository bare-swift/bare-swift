# Gate 37 Retrospective: Phase 37 → Phase 38

**Date:** 2026-05-19
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 37 (the swift-content-encoding v0.9 uniform brotli propagation anchored by [RFC-0042](../../rfcs/0042-phase-37-anchor-swift-content-encoding-v0.9-uniform-brotli-propagation.md)) against the Gate 1–36 criteria template and recommends whether Phase 38 should start. Phase 37 was a **single-tranche existing-package pure-dep-bump** (37A swift-content-encoding v0.9).

**Notable:** Phase 37 closes the **7-phase codec-tier true-memory-streaming arc** (Phase 30 → 37) — the most ambitious multi-phase project in bare-swift. The 6-instance honest-scope-under-limitation pattern from Phases 25-33 is now FULLY RESOLVED.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All anchored Tranche 37A deliverables shipped, CI green, DocC live, index bumped | ✓ PASS |
| 2 | RFC-0042 commitments fulfilled | ✓ PASS *(zero scope reshape; 21st consecutive clean RFC since Phase 17)* |
| 3 | ≥1 substantive RFC accepted *or documented absence of policy gaps* | ✓ PASS (one new pattern observation: aggregator-uniform-propagation-closes-aggregator-partial) |
| 4 | RFC-0001 conventions stress-test against Phase 37 deliverables | ✓ DONE BELOW |
| 5 | Phase 38 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0043 below) |

**Overall:** Items 1–4 pass. Item 5 is open. Phase 38 plans should be drafted but not executed until item 5 closes via RFC-0043.

Phase 37 calendar time was ~15-20 min wall-clock (within wrapper-pattern pure-dep-bump sub-bucket of 10-20 min).

---

## Item 1 — Phase 37 deliverables shipped ✓ PASS

| Tranche | Package | Tag | New tests | DocC | Index | CI |
|---|---|---|---|---|---|---|
| 37A | swift-content-encoding | **v0.9.0** | 0 (91 unchanged) | ✓ | ✓ (0.8.0→0.9.0) | green (first-try) |

Single tranche. **7-phase codec-tier true-memory-streaming arc CLOSED.**

**Phase 37 final tally:**
- 37A `swift-content-encoding v0.9.0`: swift-brotli dep 0.5 → 0.6. v0.8's partial-propagation acknowledgment retired in favor of uniform-propagation note.
- Public API surface byte-for-byte preserved. All 91 v0.8 tests pass without modification.
- ~15-20 min wall-clock — within pure-dep-bump sub-bucket.
- **First-try-clean CI streak NOW AT 8** (Phase 32A + 33A + 34A + 35A + 35B + 35C + 36A + 37A).
- **21-consecutive zero-scope-reshape RFC streak** preserved.

**Honest-scope-under-limitation pattern: FULLY RESOLVED (6/6).**
- 6 instances historically codified (Phases 25 / 28 / 30 / 31 / 32 / 33).
- All 6 resolutions logged:
  - Phase 34: deflate v0.6 (state-machine).
  - Phase 35A: gzip v0.6 (wrap inheritance).
  - Phase 35B: zlib v0.6 (wrap inheritance).
  - Phase 35C: content-encoding v0.8 (partial — deflate/gzip/zlib paths).
  - Phase 36A: brotli v0.6 (state-machine).
  - **Phase 37A: content-encoding v0.9 (uniform — brotli path; partial-propagation acknowledgment retired).**

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists **49 packages** — no count change.

---

## Item 2 — RFC-0042 commitments ✓ PASS (zero scope reshape)

[RFC-0042](../../rfcs/0042-phase-37-anchor-swift-content-encoding-v0.9-uniform-brotli-propagation.md) committed to:

| RFC-0042 commitment | Outcome |
|---|---|
| One existing package: content-encoding 0.8 → 0.9 | ✓ shipped. |
| Pure dep bump swift-brotli 0.5 → 0.6 | ✓ shipped. |
| Replace v0.8 partial-propagation note with uniform-propagation note in CHANGELOG | ✓ shipped. |
| All v0.8 tests continue to pass unchanged | ✓ all 91 tests pass. |
| DocC tagline updated | ✓ shipped. |
| Calendar estimate **deferred to brainstorm reality-check** (~10-30 min per pure-dep-bump sub-bucket) | ✓ honored. Actual ~15-20 min — mid-bracket. 16th successful reality-check application. |
| 0-2 new tests (per RFC) | ✓ shipped 0 new tests; existing test coverage is adequate. |
| Non-breaking on v0.1-v0.8 | ✓ confirmed. |
| Cross-package DocC discipline | ✓ honored — tagline update uses single-backtick for any cross-package symbol refs (none in this case; all updates were prose-level). |

**Zero scope reshape this phase.** **21st consecutive clean-from-RFC phase** (Phases 17-37).

**Estimate-vs-actual:** RFC-0042 said "~10-30 min" (sub-bucket); actual ~15-20 min (mid-bracket). 16th successful reality-check application. Pure-dep-bump sub-bucket calibration is now validated at 4 instances (35A, 35B, 35C, 37A — all 10-20 min).

---

## Item 3 — Revised criterion ✓ PASS

**Pre-Phase 37 (Gate 36 closeout):** RFC-0042 accepted 2026-05-19.

**During Phase 37 execution:** zero RFCs accepted.

Phase 37 codified one new pattern observation and reinforced existing patterns:

- **NEW pattern observation: aggregator-uniform-propagation closes aggregator-partial-propagation.** The partial-propagation-acknowledgment sub-pattern from Phase 35C is now demonstrated to have a clean follow-up: when the last upstream catches up (here: brotli v0.6 shipped in Phase 36), the aggregator (here: content-encoding) bumps its dep and the partial-propagation note is retired. The complete two-act lifecycle is now: **Act 1** ship partial with honest note while some upstream is still pending → **Act 2** ship full with retirement note once all upstreams have moved. **Memory:** future partial-propagation acknowledgments should explicitly name the Act 2 follow-up phase in the Act 1 CHANGELOG to make the closure plan visible.
- **Honest-scope-under-limitation pattern: 6 instances, 6/6 RESOLVED.** The pattern's full arc — from introduction (Phase 25) through proliferation (Phases 28-33) through resolution (Phase 34-37) — is now complete. **Memory:** the pattern is canonical procedural law for honest-scope deferrals in API-stable v0.5/v0.7 ships; the resolution lifecycle is now demonstrated end-to-end.
- **7-phase codec-tier arc closed.** The most ambitious multi-phase project in bare-swift. Demonstrates the streaming-symmetric-API-surface-ahead-of-true-streaming pattern's full lifecycle: foundation → state-machine refactors → wrapper inheritance → aggregator partial → aggregator uniform.
- **First-try-clean CI streak at 8.** Strongest sustained streak since the 19-phase Phase 16-30 streak that was broken at Phase 31A. Worth tracking whether the rebuild reaches 19.
- **Pure-dep-bump sub-bucket at 4 validated instances** (35A, 35B, 35C, 37A). Procedural law.
- **Reality-check-before-RFC-estimate at 16/16 successful applications.** Procedural law.
- **Brainstorm-empowered-by-RFC scope simplification at 5 instances** (Phase 30, 32, 33, 34, 36). Strongly canonical.

None warrant a new cross-package RFC. The aggregator-uniform-propagation observation is captured in this retrospective + the v0.9 CHANGELOG + the Phase 37A memory closure.

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

| Convention | swift-content-encoding v0.9 |
|---|---|
| Module name PascalCase | ✓ ContentEncoding |
| Apache-2.0 + LLVM exception SPDX | ✓ |
| Sendable-clean | ✓ |
| Single public error enum | ✓ ContentEncodingError (cases unchanged from v0.8) |
| Public APIs Foundation-free | ✓ |
| README + DocC updated | Partial (DocC tagline only) |
| CHANGELOG with v0.9.0 entry | ✓ uniform-propagation narrative + arc-closure note |
| CI green | ✓ first-try |
| DocC bundle | ✓ first-try |
| Sanitizers ON | ✓ |
| Backwards-compat preservation | ✓ all 91 tests pass unchanged |

### Deviations and findings

**1. README not updated.** **README-deferral-during-orchestration-sweeps at 8 instances** (Phase 27, 31, 32, 33, 34, 35, 36, 37). Fully canonical procedural acceptance. The CHANGELOG entry is the canonical adopter-facing reference.

**2. CHANGELOG narrates the 7-phase arc.** Phase 37's v0.9.0 CHANGELOG entry includes a phase-by-phase recap of the arc closure (Phase 30 → 37). This is unusual for a pure dep-bump CHANGELOG — typically brief — but appropriate given the symbolic closure value. **Memory:** arc-closing CHANGELOG entries can narrate the arc; the narrative belongs in the CHANGELOG (not just in retrospectives or memory) because adopters read CHANGELOGs.

**3. Zero new tests is the right call.** All 91 v0.8 tests exercise the brotli wrap path via `br`-containing chains. They pass unchanged on brotli v0.6 — that's the inheritance pattern working as intended. Adding new tests would be ceremony without value.

### Verdict

RFC-0001 held cleanly. One minor docs gap (README deferred — 8th instance of established pattern). The CHANGELOG narrative is appropriate for an arc-closing tranche. Zero new tests is correct.

---

## Item 5 — Phase 38 anchor decision ✗ NOT YET (recommendation)

Phase 38 anchor candidates surveyed:

| Wave | Packages | Risk | Adoption fit | Deferral count |
|---|---|---|---|---|
| **swift-oauth2-client v0.4 (active wrapper / JWKS / mTLS / etc.)** | oauth2-client v0.4 | Medium | **Highest** — v0.3 shipped Phase 29; 8 phases settle time. Picks up a new narrative (auth-tier) after codec-tier arc closed. Candidate features have concrete adopter use cases. | 6 (Phase 32-37) |
| swift-brotli v0.7 + swift-deflate v0.7 window carry | 2 packages | Medium | Low-medium — ratio improvement only. | 13 each (Phases 25-37) |
| Codec one-shot/streaming unification | swift-deflate + swift-brotli | Medium | Low — internal cleanup; no adopter-visible change. | 0 (newly viable post-Phase 36) |
| swift-distributed-tracing-bridge v0.3 (B3 + Jaeger) | bridge v0.3 | Low-medium | Low — 18 deferrals; no concrete demand. | 18 |
| swift-idna v0.3 (Bidi + ContextJ + ContextO) | swift-idna v0.3 | Medium | Low — rarely-hit Unicode rules. | 22 |
| RS-family JWT algorithms | swift-jwt-verify v0.4 | High | Low — **STILL BLOCKED** on swift-crypto `_RSA` SPI. | n/a (blocked) |
| Crypto-adjacent | various | Medium-low | Low — 29x rejected. | 29 (rejected) |
| Package rename `swift-jwt-verify` → `swift-jwt` | rename project | Low | Low-medium | n/a |

### Recommendation: **Anchor on swift-oauth2-client v0.4**

Reasons:

1. **Picks up a new narrative.** After the codec-tier arc closure (Phase 30 → 37), Phase 38 starts a fresh narrative arc rather than continuing internal cleanup. Auth-tier is the natural next theme.
2. **Highest adoption fit on the queue.** OAuth2 + OIDC adopters from Phases 20-29 have had 8 phases of settle time since v0.3. Active-wrapper / JWKS / mTLS / private_key_jwt / device-code grant are concrete features with adopter use cases.
3. **6 deferrals on the queue.** Tied with codec one-shot/streaming unification (which is internal cleanup with no adopter value). Highest among waves with adopter-visible value.
4. **Reasonable scope.** Per brainstorm reality-check, v0.4 could ship 1-2 of the candidate features (e.g., active-wrapper + JWKS as the most-demanded pair). Single-tranche feasible.
5. **Audience expansion.** Auth-tier adopters are partially different from codec-tier adopters; Phase 38 broadens the project's reach.

### Phase 38 tranche sketch (subject to formal RFC + brainstorm reality-check)

**Shape A (recommended pending reality-check): single tranche, active-wrapper + JWKS.**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 38A | swift-oauth2-client v0.4 | `OAuth2Client.activeWrapper(...)` (orchestrates token cache + refresh + retry) + JWKS support (`JWKS` struct + URL fetch via caller-provided closure + key lookup by `kid`). 8-12 new tests. | ~1.5-3 hours (pending reality-check) |

**Shape B (alternate): split active-wrapper from JWKS into 2 tranches.**

Brainstorm reality-checks based on whether the two features compose cleanly in a single API (e.g., does active-wrapper need JWKS for verification of ID tokens?). If they're naturally separable, Shape A; if entangled, Shape B.

### Why other waves were rejected for Phase 38

- **swift-brotli v0.7 + swift-deflate v0.7 window carry** — ratio improvement; not blocking; no adopter demand. Defer to Phase 39+.
- **Codec one-shot/streaming unification** — internal cleanup with no adopter-visible change. Could ride as a follow-up to Phase 38 if scope permits. Defer to Phase 39+.
- **B3 + Jaeger, idna v0.3** — long-running correct-deferrals; no concrete demand.
- **RS-family JWT** — blocked on swift-crypto `_RSA` SPI.
- **Crypto-adjacent** — 29th rejection.
- **Package rename** — premature; can ride alongside any future swift-jwt-verify version bump.

**Action:** author the anchor decision as **RFC-0043** (Phase 38 anchor: swift-oauth2-client v0.4 active wrapper + JWKS) and accept it before Phase 38 plans start.

---

## Open work to clear Gate 37

1. **Write RFC-0043** (Phase 38 anchor: swift-oauth2-client v0.4 active wrapper + JWKS). ~1-hour task.
2. **Begin Phase 38 brainstorm + reality-check** after RFC-0043 accepts. Brainstorm should:
   - Survey swift-oauth2-client v0.3 internals (TokenStorage, CachedToken, InMemoryTokenStorage, OIDCClaims).
   - Decide active-wrapper API shape: should it be a class actor, a struct holding an actor reference, or a free function? Sendable + Foundation-free constraint applies.
   - Decide JWKS API shape: caller-provided HTTP closure (matches caller-driven-HTTP pattern from v0.1/v0.2) vs. embedded fetch.
   - Decide Shape A (combined tranche) vs Shape B (separated tranches).
   - Maintain DocC discipline: cross-package refs to swift-crypto or swift-jwt-verify use single-backtick.

---

## What this retrospective changes for Phase 38

- The plan-then-execute-inline workflow stays.
- **CI first-try-clean streak at 8** (Phase 32A through 37A; rebuilding strongly after Phase 31A break; tracking whether it reaches 19).
- **Honest-scope-under-limitation pattern at 6 instances, 6/6 RESOLVED.** Pattern arc complete.
- **Brainstorm-empowered-by-RFC scope simplification at 5 instances** — firmly canonical.
- **Reality-check-before-RFC-estimate at 16/16.**
- **NEW pattern observation:** aggregator-uniform-propagation closes aggregator-partial-propagation (two-act lifecycle).
- **README-deferral-during-orchestration-sweeps at 8 instances** — fully canonical.
- **Codec-tier true-memory-streaming arc CLOSED.** Phase 38 picks up a new narrative (auth-tier).
- **Pure-dep-bump sub-bucket validated at 4 instances** (~10-20 min).

---

## Decision

Gate 37 is **PASSED on items 1, 2, 3, 4** and **NOT YET PASSED on item 5 (open)**. Phase 38 plans should be drafted but not executed until item 5 closes via RFC-0043.

Phase 37 closed the 7-phase codec-tier true-memory-streaming arc (Phase 30 → 37) via swift-content-encoding v0.9 uniform brotli propagation. The 6-instance honest-scope-under-limitation pattern from Phases 25-33 is now FULLY RESOLVED. 21-consecutive zero-scope-reshape RFC streak preserved. First-try-clean CI streak rebuilding at 8.

The next step is **RFC-0043 anchoring Phase 38 on swift-oauth2-client v0.4 active wrapper + JWKS** — picks up a new narrative (auth-tier) after the codec-tier arc closure. Codec one-shot/streaming unification, brotli/deflate v0.7 window carry, B3 + Jaeger, idna v0.3, RS-family JWT (when swift-crypto stabilizes `_RSA`), and package rename remain on the queue for Phase 39+.

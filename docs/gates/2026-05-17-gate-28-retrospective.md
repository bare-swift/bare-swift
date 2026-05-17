# Gate 28 Retrospective: Phase 28 → Phase 29

**Date:** 2026-05-17
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 28 (the swift-content-encoding v0.6 multi-coding streaming wave anchored by [RFC-0033](../../rfcs/0033-phase-28-anchor-content-encoding-v0.6-multi-coding-streaming.md)) against the Gate 1–27 criteria template and recommends whether Phase 29 should start. Phase 28 was a **single-tranche existing-package minor bump** (28A swift-content-encoding v0.6). Shipped cleanly.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All anchored Tranche 28A deliverables shipped, CI green, DocC live, index bumped | ✓ PASS |
| 2 | RFC-0033 commitments fulfilled | ✓ PASS *(zero scope reshape; 12th consecutive clean RFC since Phase 17)* |
| 3 | ≥1 substantive RFC accepted *or documented absence of policy gaps* | ✓ PASS (3 new orchestration patterns codified) |
| 4 | RFC-0001 conventions stress-test against Phase 28 deliverables | ✓ DONE BELOW |
| 5 | Phase 29 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0034 below) |

**Overall:** Items 1–4 pass. Item 5 is open. Phase 29 plans should be drafted but not executed until item 5 closes via RFC-0034.

Phase 28 calendar time was ~30 min wall-clock (squarely in wrapper-pattern bracket).

---

## Item 1 — Phase 28 deliverables shipped ✓ PASS

| Tranche | Package | Tag | New tests | DocC | Index | CI |
|---|---|---|---|---|---|---|
| 28A | swift-content-encoding | **v0.6.0** | 9 (67 total) | ✓ | ✓ (0.5.0→0.6.0) | green (first-try) |

Single tranche shipped. **Streaming HTTP body story (encode side) COMPLETE.**

**Phase 28 final tally:**
- Multi-coding streaming via cascaded `drain()` pipeline.
- v0.5's single-stage `InnerEncoder` enum replaced with multi-stage `[InnerCoding]` ordered array.
- Identity coding as buffering passthrough (uniform cascade composition).
- `.multipleCodingsNotStreamable` error case kept for backwards-compat but no longer thrown.
- ~180 source LOC + ~115 test LOC = ~295 total change.
- 67 tests across 11 suites (60 → 67; net +9: removed 2 v0.5 multi-coding-throws tests, added 9 v0.6 multi-coding-streams tests).
- ~30 min wall-clock — within RFC-0033's 30-60 min wrapper-pattern bracket.
- **17-consecutive first-try-clean CI streak** (Phases 16-28).
- **12-consecutive zero-scope-reshape RFC streak** (Phases 17-28).
- 3 codec deps bumped (gzip + zlib + brotli 0.3 → 0.4).
- v0.5 single-coding behavior preserved (regression-tested via per-coding round-trip tests).

**Streaming HTTP body story (encode side) COMPLETE:**
- ✓ Phase 22-24: brotli + deflate + gzip + zlib streaming encoders (v0.3).
- ✓ Phase 25: content-encoding v0.5 single-coding wiring.
- ✓ Phase 27: codec drain() API (v0.4) for in-stream composition.
- ✓ **Phase 28: content-encoding v0.6 multi-coding pipeline.**

**Decode side remains one-shot only** — streaming decoders sweep is Phase 29+ candidate.

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists **49 packages** — no count change.

---

## Item 2 — RFC-0033 commitments ✓ PASS (zero scope reshape)

[RFC-0033](../../rfcs/0033-phase-28-anchor-content-encoding-v0.6-multi-coding-streaming.md) committed to:

| RFC-0033 commitment | Outcome |
|---|---|
| Existing package swift-content-encoding v0.5 → v0.6 (additive minor bump) | ✓ shipped. |
| Multi-coding streaming via cascaded drain() pipeline | ✓ shipped. |
| InnerEncoder enum → `[InnerCoding]` array | ✓ shipped. |
| Identity coding as buffering passthrough | ✓ shipped (composes uniformly). |
| `.multipleCodingsNotStreamable` case kept for backwards-compat, no longer thrown | ✓ shipped. |
| Pipeline semantics LOCKED at RFC time (left-to-right per RFC 9110 § 8.4; cascade via drain in update; cascade via update-then-finish in finish) | ✓ all tranches honor locked semantics; tests verify byte-equality with one-shot decode. |
| swift-gzip/zlib/brotli dep bumps 0.3 → 0.4 | ✓ all 3 bumped. |
| ~15 tests (RFC range) | ✓ 9 new tests (slightly below lower bound but full coverage of multi-coding combinations). |
| v0.5 single-coding preserved byte-for-byte | ✓ regression-tested. |
| Per-tranche calendar **deferred to brainstorm reality-check** | ✓ honored. Reality-check confirmed cascade semantics upfront. Estimate locked at 30-60 min; actual ~30 min. |

**Zero scope reshape this phase.** **12th consecutive clean-from-RFC phase** (Phases 17-28).

**Test count below lower-bound but coverage complete:** RFC said 10-15 new tests; shipped 9 new + 2 removed v0.5 throws-tests = net +7. Coverage matrix is comprehensive (2-coding × 3 orderings + 3-coding × 1 + identity-in-chain × 2 + multi-chunk + unsupported-in-pipeline + backwards-compat-enum-case-pattern-match = 9 tests covering 9 distinct scenarios). **Memory:** test count matters less than coverage matrix.

**Estimate-vs-actual:** RFC-0033 said "30-60 min pending reality-check"; actual ~30 min. Lower bound. **Wrapper-package calibration holds at 7 instances now.**

---

## Item 3 — Revised criterion ✓ PASS

**Pre-Phase 28 (Gate 27 closeout):** RFC-0033 accepted 2026-05-17.

**During Phase 28 execution:** zero RFCs accepted.

Phase 28 surfaced **three new canonical patterns** + reinforced existing patterns:

- **NEW: Cascaded-pipeline orchestration pattern (Phase 28 codification).** For orchestrating multiple streaming primitives in a chain: array-of-stages + per-stage update/drain/finish methods on inner enum + cascade loop in outer update/finish. The key idiom is "feed final-of-stage-i to update-of-stage-i+1; stage-i+1's finish flushes those bytes naturally along with its terminator." Applied at content-encoding v0.6; reusable for any future orchestration layer that composes multiple streaming primitives (hypothetical HTTP chunked-encoding wrapper, multi-format streaming serializer).
- **NEW: Identity-as-buffering-passthrough pattern (Phase 28 codification).** When an orchestration enum needs a no-op case to compose with mutating-state cases, model the no-op as a buffering accumulator with the same update/drain/finish API surface. Avoids special-casing the cascade loop. Applied to identity coding in v0.6 pipeline.
- **NEW: Backwards-compat-via-keep-enum-case-but-stop-throwing pattern (Phase 28 codification).** When a feature-gate error case becomes obsolete because the underlying gate has opened (e.g., the limitation it documented no longer applies), keep the case in the enum so downstream callers pattern-matching it continue to compile. Document as "deprecated but defined for backwards-compat." Applied to `.multipleCodingsNotStreamable` in v0.6.
- **Honest-scope-under-limitation pattern graduated.** Phase 25 codified the pattern (multi-coding throws-with-documented-future-path). Phase 28 closed that future path. **Memory:** the pattern is sound for limitation-driven scope decisions: ship honest now, document the path, complete the work when the underlying gate opens. Track from Phase 25 → Phase 28 as a 1-instance complete arc.
- **Reality-check-before-RFC-estimate at 7/7 successful applications.** Phase 28's reality-check confirmed cascade semantics upfront; zero scope reshape.
- **Wrapper-package calibration at 7 instances** (Phases 23-25, 27 × 4, 28). Pattern is procedural law.

None warrant a new cross-package RFC. **Documented absence of policy gaps. Criterion satisfied.**

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

| Convention | swift-content-encoding v0.6 |
|---|---|
| Module name PascalCase | ✓ ContentEncoding |
| Apache-2.0 + LLVM exception SPDX | ✓ |
| Sendable-clean | ✓ all new types Sendable |
| Single public error enum | ✓ ContentEncodingError (5 cases; unchanged from v0.5) |
| Public APIs Foundation-free | ✓ |
| README + DocC updated | ✓ |
| CHANGELOG with v0.6.0 entry | ✓ |
| CI green | ✓ first-try |
| DocC bundle | ✓ |
| Sanitizers ON | ✓ |
| Backwards-compat preservation | ✓ v0.5 single-coding byte-for-byte; `.multipleCodingsNotStreamable` enum case kept |

### Deviations and findings

**1. Three new canonical patterns codified in one phase.** Phase 28 surfaced three reusable orchestration patterns. Cascaded-pipeline orchestration is the meaningful "big" one; identity-as-buffering-passthrough and backwards-compat-via-keep-enum-case-but-stop-throwing are smaller technique-level patterns. **Memory:** when a phase codifies multiple patterns, it's worth highlighting in the gate retro for future-Claude pattern recognition.

**2. Test removal during scope change.** Phase 28 removed 2 v0.5 multi-coding-throws tests (the throws no longer happens). Test removal is unusual but appropriate when the feature being tested is intentionally removed/changed. **Memory:** clearly document test removals in the commit message + CHANGELOG so the change history is traceable.

**3. Streaming HTTP body story (encode side) feature-complete.** No remaining encode-side deferrals. Phase 28 closes the 6-phase arc (22-28) cleanly. Future encode-side work would be ratio improvements (window carry — Phase 29+) or new codec algorithms (zstd, etc. — not on roadmap).

### Verdict

RFC-0001 held cleanly under Phase 28 stress. Three new pattern codifications + one pattern arc (honest-scope-under-limitation → graduate) completed.

---

## Item 5 — Phase 29 anchor decision ✗ NOT YET (recommendation)

Phase 29 anchor candidates surveyed:

| Wave | Packages | Risk | Adoption fit | Deferral count |
|---|---|---|---|---|
| **swift-oauth2-client v0.3 (token caching + refresh + OIDC ID-token helpers)** | swift-oauth2-client v0.3 | Medium | **High** — closes documented v0.2 deferrals; ~1.5 days v0.2 settle time; clear adopter value (every OIDC adopter needs caching + auto-refresh). | 2 (Phases 27, 28) |
| Streaming decoders sweep (brotli + deflate + gzip + zlib v0.5 inflate) | 4 packages, 4 tranches | High (3-5hr per tranche; major refactor of Decoder.swift / Inflater.swift state machines) | High — would complete streaming HTTP body story bidirectionally. | 2 each (Phases 27, 28) |
| swift-brotli v0.5 + swift-deflate v0.5 window carry across chunks | 2 packages | Medium | Low-medium — compression ratio improvements. | 4 each (Phases 25-28) |
| Single-codec streaming decode (e.g., swift-deflate v0.5 inflate only) | 1 package | Medium-high (3-5hr) | Medium — starts streaming-decode sweep but doesn't complete it. | 1 |
| swift-distributed-tracing-bridge v0.3 (B3 + Jaeger) | bridge v0.3 | Low-medium | Low — 9 deferrals; no concrete demand. | 9 |
| swift-idna v0.3 (Bidi + ContextJ + ContextO) | swift-idna v0.3 | Medium | Low — rarely-hit Unicode rules. | 13 |
| RS-family JWT algorithms | swift-jwt-verify v0.4 | High | Low — **STILL BLOCKED** on swift-crypto `_RSA` SPI. | n/a (blocked) |
| Crypto-adjacent | various | Medium-low | Low — 24x rejected. | 24 (rejected) |
| Package rename `swift-jwt-verify` → `swift-jwt` | rename project | Low | Low-medium | n/a |

### Recommendation: **Anchor on swift-oauth2-client v0.3 (token caching + refresh + OIDC ID-token helpers)**

Reasons:

1. **Closes documented v0.2 deferrals.** v0.1 + v0.2 CHANGELOGs explicitly defer "token caching / lifecycle / auto-refresh" and "OIDC-specific helpers (ID-token verification composes with swift-jwt-verify)" to v0.3+. Both are concrete features with clear adopter value.
2. **v0.2 settle time complete.** v0.2 shipped ~1.5 days ago; no incidents, no breaking-API feedback. Stable base for v0.3.
3. **Strong adopter value.** Every OIDC adopter wants:
   - **Token caching** so they don't re-do the token exchange on every request.
   - **Auto-refresh** when access tokens expire.
   - **ID-token claim helpers** for the OIDC half of OAuth.
   v0.1 + v0.2 + manual composition can do all this — but it's ergonomic-painful. v0.3 makes it library-easy.
4. **Modest scope** (~1-2 hours).
5. **No new external deps.** Composes with existing swift-jwt-verify (already in some adopters' graphs) + swift-bytes. Token storage is a protocol caller-implements (no swift-keychain or storage-backend dep).
6. **Audience continuity.** Same auth/OIDC adopters from Phases 11/19/20/21/26.
7. **Reality-check applies.** Brainstorm should survey swift-jwt-verify's `JWTVerifier.verify(_:) -> Verified` interface to confirm ID-token helper composition; settle TokenStorage protocol design (in-memory default? caller protocol?); settle refresh strategy semantics (return cached vs refresh-and-cache); settle thread-safety (Sendable storage protocol vs single-task usage).
8. **Non-breaking additive minor bump.** All v0.2 APIs unchanged. v0.3 adds:
   - `TokenStorage` protocol (Sendable).
   - `OAuth2Client.TokenCache` struct or `OAuth2Client.RefreshingClient` wrapper — TBD by brainstorm.
   - `OAuth2Client.idTokenClaims(_:)` helper or `OAuth2Client.OIDCTokenResponse` that surfaces ID token + verified claims. Composes with swift-jwt-verify externally.

### Phase 29 tranche sketch (subject to formal RFC + brainstorm reality-check)

**Shape A (recommended pending reality-check): single tranche, token caching + refresh + OIDC helpers.**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 29A | swift-oauth2-client v0.3 | TokenStorage protocol + TokenCache/RefreshingClient wrapper + ID-token claim composition + ~15-20 tests | ~1-2 hours (pending reality-check) |

**Shape B (alternate): two-tranche if scope splits cleanly.**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 29A | swift-oauth2-client v0.3 | Token caching + auto-refresh | ~1 hour |
| 29B | swift-oauth2-client v0.4 | OIDC ID-token claim helpers | ~30-60 min |

Shape A is preferred — both feature sets compose well in one minor bump.

### Why other waves were rejected for Phase 29

- **Streaming decoders sweep** — 12-20hr coordinated sweep is the biggest investment of any candidate; adopter demand unclear (decode-side use cases are rare; one-shot decode handles most HTTP responses). **Phase 30+ candidate** — re-examine when streaming-decode demand surfaces from real adopter feedback.
- **swift-brotli v0.5 + swift-deflate v0.5 window carry** — ratio improvements; not blocking any adopter. Defer.
- **Single-codec streaming decode** — starts the sweep without completing it; bundle the sweep when it's prioritized.
- **B3 + Jaeger, idna v0.3** — no demand.
- **RS-family JWT** — blocked on `_RSA` SPI.
- **Crypto-adjacent** — 24th rejection.
- **Package rename** — premature; Phase 30+ if at all.

**Action:** author the anchor decision as **RFC-0034** (Phase 29 anchor: swift-oauth2-client v0.3 token caching + refresh + OIDC helpers) and accept it before Phase 29 plans start.

---

## Open work to clear Gate 28

1. **Write RFC-0034** (Phase 29 anchor: swift-oauth2-client v0.3 token caching + refresh + OIDC helpers). 1-hour task.
2. **Begin Phase 29 brainstorm + reality-check** after RFC-0034 accepts. Brainstorm should:
   - Survey swift-jwt-verify's verify path — confirm ID-token verification composes cleanly.
   - Settle TokenStorage protocol shape (in-memory default? Sendable for cross-task use? methods: load / store / delete?).
   - Settle refresh strategy semantics (refresh-before-expiry threshold? auto-refresh on expired-error response?).
   - Settle whether v0.3 ships `RefreshingClient` wrapper or extends `OAuth2Client` directly.
   - Settle thread-safety story for token cache state.

---

## What this retrospective changes for Phase 29

- The plan-then-execute-inline workflow stays.
- **Wrapper-package calibration confirmed at 7 instances.** Phase 29 is **not** a wrapper-pattern phase — it ships net-new features (TokenStorage, RefreshingClient, OIDC helpers). Expect 1-2 hours per RFC, not 30-60 min.
- **Three new patterns codified** (cascaded-pipeline orchestration; identity-as-buffering-passthrough; backwards-compat-via-keep-enum-case-but-stop-throwing). Available for future orchestration work.
- **Reality-check-before-RFC-estimate at 7/7** — Phase 29 RFC defers per-tranche estimate to brainstorm.
- **Streaming HTTP body story (encode side) COMPLETE.** Future encode-side work is ratio improvements (window carry) or new codec algorithms.
- **Honest-scope-under-limitation arc completed** Phase 25 → Phase 28. Pattern validated.
- **Carried forward:** Phase 28 left no documented deferrals beyond v0.7+ candidates already listed (streaming decode, reset, flush, Level→brotli Quality mapping).

---

## Decision

Gate 28 is **PASSED on items 1, 2, 3, 4** and **NOT YET PASSED on item 5 (open)**. Phase 29 plans should be drafted but not executed until item 5 closes via RFC-0034.

Phase 28 completed the streaming HTTP body story (encode side). 17-consecutive first-try-clean CI streak; 12-consecutive zero-scope-reshape RFC streak. Three new orchestration patterns codified.

The next step is **RFC-0034 anchoring Phase 29 on swift-oauth2-client v0.3 (token caching + refresh + OIDC ID-token helpers)** — closes documented v0.2 deferrals; completes the OAuth 2.0 / OIDC client story for typical adopter use cases. Streaming decoders sweep, swift-brotli v0.5 + swift-deflate v0.5 window carry, B3 + Jaeger, idna v0.3, RS-family JWT (when swift-crypto stabilizes `_RSA`), and package rename remain on the queue for Phase 30+.

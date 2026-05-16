# Gate 25 Retrospective: Phase 25 → Phase 26

**Date:** 2026-05-16
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 25 (the swift-content-encoding v0.5 single-coding streaming wave anchored by [RFC-0030](../../rfcs/0030-phase-25-anchor-swift-content-encoding-v0.5-streaming.md)) against the Gate 1–24 criteria template and recommends whether Phase 26 should start. Phase 25 was a **single-tranche existing-package minor bump** (25A swift-content-encoding v0.5). Shipped cleanly.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All anchored Tranche 25A deliverables shipped, CI green, DocC live, index bumped | ✓ PASS |
| 2 | RFC-0030 commitments fulfilled | ✓ PASS *(zero scope reshape; 9th consecutive clean RFC since Phase 17)* |
| 3 | ≥1 substantive RFC accepted *or documented absence of policy gaps* | ✓ PASS (canonical-pattern reinforcement + 1 new sub-pattern codified) |
| 4 | RFC-0001 conventions stress-test against Phase 25 deliverables | ✓ DONE BELOW |
| 5 | Phase 26 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0031 below) |

**Overall:** Items 1–4 pass. Item 5 is open. Phase 26 plans should be drafted but not executed until item 5 closes via RFC-0031.

Phase 25 calendar time was ~45 min wall-clock — within Gate 24's "30-60 min per tranche for wrapper packages" calibration.

---

## Item 1 — Phase 25 deliverables shipped ✓ PASS

| Tranche | Package | Tag | New code | DocC | Index | CI |
|---|---|---|---|---|---|---|
| 25A | swift-content-encoding | **v0.5.0** (existing; minor bump) | ~120 source LOC + ~270 test LOC + ~50 docs LOC | ✓ | ✓ (0.4.0→0.5.0) | green (first-try) |

Single tranche shipped. **Wires codec-tier streaming sweep (Phase 22-24) through the HTTP `Content-Encoding` layer** for the 95%+ single-coding case. Multi-coding chains throw at init; multi-coding streaming awaits Phase 26+ codec-tier `drain()` API.

**Phase 25 final tally:**
- `ContentEncoding.Streaming.Encoder(contentEncoding:level:) throws` with InnerEncoder dispatch enum (identity-accumulator / Gzip.Streaming.Encoder / Zlib.Streaming.Encoder / Brotli.Streaming.Encoder).
- 2 new `ContentEncodingError` cases: `.multipleCodingsNotStreamable(String)`, `.encoderFinished`.
- 18 new tests across 1 new suite (60 total tests; up from 42); first-try CI green.
- ~45 min calendar wall-clock — within wrapper-package calibration.
- **11-consecutive first-try-clean CI streak** (Phases 16-25).
- **9-consecutive zero-scope-reshape RFC streak** (Phases 17-25).
- swift-gzip / swift-zlib / swift-brotli deps bumped 0.2.0 → 0.3.0.

**Codec-tier streaming sweep WIRED through HTTP layer (single-coding):**
- ✓ Phase 22A swift-brotli v0.3 streaming
- ✓ Phase 23A swift-deflate v0.3 streaming
- ✓ Phase 24A swift-gzip v0.3 streaming
- ✓ Phase 24B swift-zlib v0.3 streaming
- ✓ Phase 25A swift-content-encoding v0.5 single-coding streaming
- 🟡 Phase 26+: codec-tier `drain()` API → unblocks multi-coding in content-encoding v0.6

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists **49 packages** — no count change.

---

## Item 2 — RFC-0030 commitments ✓ PASS (zero scope reshape)

[RFC-0030](../../rfcs/0030-phase-25-anchor-swift-content-encoding-v0.5-streaming.md) committed to:

| RFC-0030 commitment | Outcome |
|---|---|
| Existing package swift-content-encoding v0.4 → v0.5 (additive minor bump) | ✓ shipped. |
| `ContentEncoding.Streaming.Encoder` (init throws / update / finish throws) | ✓ shipped. |
| Multi-coding throws `.multipleCodingsNotStreamable(header)` at init | ✓ shipped. |
| Identity coding buffers internally; returns at finish | ✓ shipped. |
| gzip/x-gzip/deflate/x-deflate/br dispatch to inner streaming encoders | ✓ shipped. |
| 2 new error cases (`.multipleCodingsNotStreamable`, `.encoderFinished`) | ✓ shipped. |
| Dep bumps: swift-gzip + swift-zlib + swift-brotli 0.2 → 0.3 | ✓ shipped. swift-deflate dep unchanged (not directly used). |
| ~15-20 tests (RFC range) | ✓ shipped 18 tests (mid-bracket). |
| Out of scope: multi-coding streaming, streaming decode, reset(), flush, Level→brotli Quality mapping | ✓ all honored. |

**Zero scope reshape this phase.** **9th consecutive clean-from-RFC phase** (Phases 17-25).

**Estimate-vs-actual:** RFC-0030 said "1-2 hours pending reality-check"; actual ~45 min. Within bracket at the low end. The "30-60 min per tranche for wrapper-pattern packages" calibration codified in Gate 24 is holding firm — Phase 25 is the 4th wrapper-pattern phase (24A gzip, 24B zlib, 25A content-encoding) all in the same time bracket.

---

## Item 3 — Revised criterion ✓ PASS

**Pre-Phase 25 (Gate 24 closeout):** RFC-0030 accepted 2026-05-16.

**During Phase 25 execution:** zero RFCs accepted.

Phase 25 surfaced one new sub-pattern + reinforced others:

- **NEW: InnerEncoder dispatch enum pattern (Phase 25 codification).** For orchestration layers wrapping multiple streaming primitives, use an enum with associated-value streaming encoders + reassign-after-mutating idiom:
  ```swift
  switch inner {
  case .gzip(var enc):
      enc.update(chunk)
      inner = .gzip(enc)
  // ...
  }
  ```
  Cleanest Swift idiom for value-type streaming dispatch. COW handles buffer efficiency. **Memory:** apply to any future wrapping streaming layer (could be useful in Phase 26 if codec-tier drain API ships).
- **Reality-check-before-RFC-estimate at 4/4 instances.** Phase 25's reality-check found multi-coding limitation upfront, allowing RFC-0030 to scope honestly. Procedural law.
- **Streaming-encoder shape applied at HTTP orchestration layer.** Same init/update/finish + State enum + double-finish-throws + update-after-finish-no-op shape works for composing inner encoders. Pattern transcends the codec primitives.
- **Honest scope under known limitation.** Multi-coding streaming would have been dishonest given the underlying API constraint; explicit throw + documented v0.6+ path is the correct compromise. Worth tracking as a meta-pattern: "ship what's honest now, document the path to the rest."
- **Exhaustive-switch-over-Error in tests at 6 instances now.** Reliable pattern reinforced.
- **DocC clean on first try** for Phase 25 — no cross-package symbol issues (Streaming.swift uses module imports + types directly without DocC cross-refs to outer types).

None warrant a new cross-package RFC. **Documented absence of policy gaps. Criterion satisfied.**

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

| Convention | swift-content-encoding v0.5 |
|---|---|
| Module name PascalCase | ✓ ContentEncoding (unchanged) |
| Apache-2.0 + LLVM exception SPDX | ✓ |
| Sendable-clean by default | ✓ Encoder is Sendable struct |
| Strict concurrency in CI | ✓ |
| Single public error enum, typed throws | ✓ ContentEncodingError (5 cases) |
| Public APIs Foundation-free | ✓ |
| README tagline + ≤30-line example | ✓ updated (streaming + multi-coding limitation note) |
| CHANGELOG with v0.5.0 entry | ✓ |
| CI green on macOS + Linux | ✓ first-try |
| DocC bundle | ✓ |
| Sanitizers ON | ✓ |
| Existing-encoder-preserved-verbatim | ✓ v0.4 encode/decode untouched |

### Deviations and findings

**1. New InnerEncoder dispatch enum pattern.** Phase 25 introduced this orchestration idiom for wrapping multiple streaming primitives. Pattern works cleanly; would likely re-emerge for any layer wrapping the codec sweep (e.g., a future HTTP framework streaming response body wrapper). **Memory:** codify as canonical pattern for future Phase 26+ orchestration layers.

**2. Honest single-coding scope.** Phase 25 ships v0.5 single-coding streaming with explicit throw on multi-coding — not silent fallback, not buffering-defeats-streaming. This is the correct approach when API constraints don't allow honest composition. **Memory:** for future phases where a feature has a known limitation (e.g., streaming decode, drain API), explicit throw + documented future path beats fudging the implementation.

**3. Brotli streaming integration with `level` mismatch.** swift-brotli's `Quality` doesn't 1:1 map to `Deflate.Encoder.Level`. v0.5 streaming `br` ignores `level` parameter (uses `.default` Quality), matching v0.4 one-shot behavior. **Memory:** the inconsistency exists in v0.4 already; v0.6 could introduce a `ContentEncoding.Streaming.Encoder.Level` enum that maps to both Deflate.Level and Brotli.Quality. Phase 27+ refinement.

**4. Multi-coding limitation documented in CHANGELOG + README + DocC.** Adopters see the constraint clearly. No surprise.

### Verdict

RFC-0001 held cleanly under Phase 25 stress. New InnerEncoder dispatch enum pattern is the only meaningful codification this gate.

---

## Item 5 — Phase 26 anchor decision ✗ NOT YET (recommendation)

Phase 26 anchor candidates surveyed:

| Wave | Packages | Risk | Adoption fit | Deferral count |
|---|---|---|---|---|
| **swift-oauth2-client v0.2 (auth URL builder + PKCE generation + state/nonce + ClientAuthMethod)** | swift-oauth2-client v0.2 | Medium (adds swift-crypto dep) | **High** — closes documented v0.2 deferrals from Phase 20; completes OAuth 2.0 client coverage (auth flow start, not just token exchange); ~3 days v0.1 settle time. | 5 (Phases 21-25) |
| Codec-tier v0.4 drain() API sweep (brotli + deflate + gzip + zlib v0.4) | brotli v0.4, deflate v0.4, gzip v0.4, zlib v0.4 | High (4-package coordinated sweep) | Medium — would unblock multi-coding streaming in content-encoding v0.6; multi-coding HTTP is rare (~1% of real traffic). | 1 each (just deferred from Phase 25) |
| Streaming decoders sweep | brotli + deflate + gzip + zlib v0.4 decode | High | Medium — symmetric to encode streaming; covers large-response-body decode use cases. | 0 (not yet deferred) |
| swift-brotli v0.4 + swift-deflate v0.4 (window carry only, no drain) | brotli v0.4, deflate v0.4 | Medium | Low-medium — ratio improvements; v0.3 just shipped. | 1 each |
| swift-distributed-tracing-bridge v0.3 (B3 + Jaeger propagation) | bridge v0.3 | Low-medium | Low — no concrete demand; 6 deferrals. | 6 |
| swift-idna v0.3 (Bidi + ContextJ + ContextO) | swift-idna v0.3 | Medium | Medium — closes Phase 15's documented deferral. | 10 |
| RS-family JWT algorithms | swift-jwt-verify v0.4 | High | Low — **STILL BLOCKED** on swift-crypto `_RSA` SPI. | n/a (blocked) |
| Crypto-adjacent | various | Medium-low | Low — 21x rejected. | 21 (rejected) |
| Package rename `swift-jwt-verify` → `swift-jwt` | rename project | Low | Low-medium | n/a |

### Recommendation: **Anchor on swift-oauth2-client v0.2 (auth URL builder + PKCE generation + state/nonce + ClientAuthMethod)**

Reasons:

1. **Closes documented v0.2 deferrals from Phase 20.** v0.1's CHANGELOG explicitly lists "authorization URL building," "PKCE verifier/challenge generation," "state / nonce generation," and "ClientAuthMethod knob" as out-of-scope-for-v0.1, deferred-to-v0.2+. All four can ship in Phase 26.
2. **Completes auth-tier coverage.** v0.1 handles the **token endpoint** (the back half of OAuth 2.0); v0.2 covers the **auth flow start** (the front half). Together, swift-oauth2-client becomes a complete OAuth 2.0 / OIDC relying-party stack alongside swift-jwt-verify (ID-token verification) + swift-bearer (bearer-token consumption) + swift-basic-auth (client-credentials Basic auth).
3. **Strong adoption demand signal.** Every OIDC adopter needs auth URL building + PKCE — these are the most-requested OAuth client features in practice. v0.1's "compose with swift-crypto externally" workaround is fine for sophisticated callers; v0.2 makes it ergonomic for the typical adopter.
4. **v0.1 settle time well-complete.** v0.1 shipped ~3 days ago; no incidents, no breaking-API feedback. Stable base for v0.2.
5. **swift-crypto dep addition is justified.** v0.1 was Foundation-free / swift-crypto-free. v0.2 adds swift-crypto for SHA256 (PKCE challenge) + secure random (state/nonce). This is **the first time swift-oauth2-client touches swift-crypto**, but the dep is necessary for actual PKCE/state generation — without it, the "generation" feature is just docs pointing at swift-crypto.
6. **Audience continuity.** Same auth/security adopters from Phases 11/19/20/21.
7. **Non-breaking additive minor bump.** All v0.1 APIs unchanged. v0.2 adds:
   - `OAuth2Client.AuthorizationURL` builder (composes auth-endpoint URL + query params).
   - `OAuth2Client.PKCE` namespace with verifier/challenge generators.
   - `OAuth2Client.State` + `OAuth2Client.Nonce` generators (or simpler `randomURLSafeToken(_:)` helper).
   - `ClientAuthMethod` enum (`.body` (default; v0.1 behavior) / `.basic` (HTTP Basic header)) + threaded through `OAuth2Client` init or per-call.
8. **Reality-check applies.** Brainstorm reality-checks swift-crypto's SHA256 + random APIs (both well-known stable APIs) + URL-encoding patterns (v0.1's FormEncoder is reusable for query-string emit).

### Phase 26 tranche sketch (subject to formal RFC + brainstorm reality-check)

**Shape A (recommended pending reality-check): single tranche, all four v0.2 features.**

| Tranche | Package | Scope | Estimated calendar (reality-check-locked) |
|---|---|---|---|
| 26A | swift-oauth2-client v0.2 | `AuthorizationURL` builder + `PKCE` verifier/challenge + state/nonce generators + `ClientAuthMethod` enum + ~25-30 tests | ~1-2 hours (pending reality-check) |

**Shape B (alternate): two tranches if scope is heavier than expected.**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 26A | swift-oauth2-client v0.2 | `AuthorizationURL` builder + `ClientAuthMethod` enum (no crypto) | ~30-60 min |
| 26B | swift-oauth2-client v0.3 | PKCE generators + state/nonce generators (adds swift-crypto dep) | ~1 hour |

Decision deferred to brainstorm phase. Per the canonical reality-check pattern, expect Shape A unless brainstorm uncovers swift-crypto API surprises or scope creep.

### Why other waves were rejected for Phase 26

- **Codec-tier v0.4 drain API sweep** — would unblock multi-coding streaming in content-encoding v0.6, but multi-coding HTTP is ~1% of real traffic. Coordinated 4-package sweep is heavier than oauth2-client v0.2. Phase 27+ candidate when multi-coding demand surfaces from adopters.
- **Streaming decoders sweep** — symmetric to encode streaming. Phase 27+ candidate; adopter demand for streaming decode is unclear in bare-swift's current adopter survey.
- **swift-brotli v0.4 + swift-deflate v0.4 window carry only** — could pair with drain sweep in a coordinated v0.4 push; premature to ship in isolation.
- **swift-distributed-tracing-bridge v0.3 (B3 + Jaeger)** — no concrete demand.
- **swift-idna v0.3** — rarely-hit rules; no demand.
- **RS-family JWT** — **still BLOCKED** on swift-crypto's `_RSA` SPI.
- **Crypto-adjacent** — 21st consecutive rejection.
- **Package rename** — premature.

**Action:** author the anchor decision as **RFC-0031** (Phase 26 anchor: swift-oauth2-client v0.2 — auth flow start) and accept it before Phase 26 plans start.

---

## Open work to clear Gate 25

1. **Write RFC-0031** (Phase 26 anchor: swift-oauth2-client v0.2). 1-hour task.
2. **Begin Phase 26 brainstorm + reality-check** after RFC-0031 accepts. Brainstorm 26A should:
   - Survey swift-crypto's `SHA256` API (stable) + `RandomNumberGenerator` / `SymmetricKey` for state/nonce random generation.
   - Verify v0.1's `FormEncoder` is reusable for query-string emit on the AuthorizationURL builder (same `application/x-www-form-urlencoded` syntax).
   - Settle on AuthorizationURL builder API shape: returns a String? a Bytes? a URL-typed value (none in bare-swift)? Likely a String for ergonomic redirect handling.
   - Settle on PKCE API shape: `PKCE.generateVerifier(length:)` + `PKCE.challenge(for:method:)` with `method: PKCEMethod = .S256`.
   - Settle on state/nonce shape: `OAuth2Client.randomToken(byteCount:)` or per-purpose `State.generate()` / `Nonce.generate()`.
   - Settle on `ClientAuthMethod` placement: per-`OAuth2Client` config OR per-call argument.

---

## What this retrospective changes for Phase 26

- The plan-then-execute-inline workflow stays.
- **Wrapper-package calibration (30-60 min per tranche)** confirmed at 4 instances now (gzip, zlib, content-encoding plus borderline brotli/deflate). Phase 26's swift-oauth2-client v0.2 is **not** a wrapper package — it ships net-new features (URL builder, PKCE generators). Expect 1-2 hours per Phase 26 RFC estimate, not 30 min.
- **InnerEncoder dispatch enum pattern (NEW)** codified — apply to future orchestration layers.
- **Reality-check-before-RFC-estimate at 4/4** — Phase 26 RFC should defer per-tranche estimate to brainstorm.
- **Honest-scope-under-limitation pattern** — Phase 25 codified that explicit throws + documented future path beats silent fallback. Apply if Phase 26 surfaces similar known-limitation cases.
- **Carried forward:** Phase 25 left explicit deferrals (multi-coding streaming → Phase 26+ drain sweep, streaming decode → Phase 27+, Level→Quality mapping → v0.6+).

---

## Decision

Gate 25 is **PASSED on items 1, 2, 3, 4** and **NOT YET PASSED on item 5 (open)**. Phase 26 plans should be drafted but not executed until item 5 closes via RFC-0031.

Phase 25 completed the single-coding HTTP-layer streaming wiring (codec-tier sweep is now adopter-usable for the 95%+ case). Streaming-encoder canonical shape composes through orchestration layers via the InnerEncoder dispatch enum pattern.

The next step is **RFC-0031 anchoring Phase 26 on swift-oauth2-client v0.2 (auth flow start — auth URL builder + PKCE generation + state/nonce + ClientAuthMethod)** — completes auth-tier coverage by closing the documented v0.2 deferrals from Phase 20. Codec-tier v0.4 drain API sweep, streaming decoders, swift-brotli v0.4 / swift-deflate v0.4 window carry, B3 + Jaeger, idna v0.3, RS-family JWT (when swift-crypto stabilizes `_RSA`), and package rename remain on the queue for Phase 27+.

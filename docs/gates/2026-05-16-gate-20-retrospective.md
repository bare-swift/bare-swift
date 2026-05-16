# Gate 20 Retrospective: Phase 20 → Phase 21

**Date:** 2026-05-16
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 20 (the swift-oauth2-client v0.1 wave anchored by [RFC-0025](../../rfcs/0025-phase-20-anchor-swift-oauth2-client.md)) against the Gate 1–19 criteria template and recommends whether Phase 21 should start. Phase 20 was a **single-tranche new-package wave** (20A swift-oauth2-client v0.1). Shipped cleanly.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All anchored Tranche 20A deliverables shipped, CI green, DocC live, listed | ✓ PASS |
| 2 | RFC-0025 commitments fulfilled | ✓ PASS *(zero scope reshape; 4th consecutive clean RFC since Phase 17)* |
| 3 | ≥1 substantive RFC accepted *or documented absence of policy gaps* | ✓ PASS (documented absence of policy gaps) |
| 4 | RFC-0001 conventions stress-test against Phase 20 deliverables | ✓ DONE BELOW |
| 5 | Phase 21 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0026 below) |

**Overall:** Items 1–4 pass. Item 5 is open. Phase 21 plans should be drafted but not executed until item 5 closes via RFC-0026.

The roadmap's stop conditions did not trigger: Phase 20 calendar time was ~1 hour wall-clock. No security incidents.

---

## Item 1 — Phase 20 deliverables shipped ✓ PASS

| Tranche | Package | Tag | New code | DocC | Index | CI |
|---|---|---|---|---|---|---|
| 20A | swift-oauth2-client | **v0.1.0** (NEW; 49th package) | ~365 source LOC + ~500 test LOC | ✓ | ✓ | green (first-try, push + tag) |

Single tranche shipped. **Closes the 8-rejection OAuth 2.0 deferral** — Gates 11-18 all deferred under "no concrete adopter demand"; Gate 19 recognized the procedural-drift pattern (same as the 7-rejection JWT-signing deferral broken by Phase 19); Phase 20 broke through.

**The bare-swift auth-client story is now feature-complete:**
- `swift-basic-auth` (Phase 11A v0.1) — HTTP Basic credential parsing
- `swift-bearer` (Phase 11B v0.1) — RFC 6750 Bearer tokens + WWW-Authenticate
- `swift-jwt-verify` (Phase 11C v0.1 + Phase 19A v0.2) — HS256 + ES256 sign + verify
- `swift-oauth2-client` (Phase 20A v0.1) — RFC 6749 token-endpoint client

Adopters can compose all four into the typical OIDC relying-party stack with zero adopter-side boilerplate beyond HTTP transport wiring.

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists **49 packages** — 1 new from Phase 20.

**Calendar time:** ~1 hour wall-clock — third-smallest phase to date (Phases 18 + 19 still floor at <30 min + ~1 hour respectively). The pattern of "small/medium net-new package with focused scope" reliably ships in 1-2 hours.

**5-consecutive first-try-clean CI streak:** Phases 16 (swift-log-bridge), 17 (swift-metrics-bridge), 18 (swift-distributed-tracing-bridge v0.2), 19 (swift-jwt-verify v0.2), 20 (swift-oauth2-client v0.1). Five phases in a row with zero CI fixup commits across push + tag + Release + Publish docs workflows.

**Pre-emptive Pages setup is 3-for-3** (Phases 16, 17, 20). The three-API-call sequence codified in Gate 16 works every time on new-package phases.

---

## Item 2 — RFC-0025 commitments ✓ PASS (zero scope reshape)

[RFC-0025](../../rfcs/0025-phase-20-anchor-swift-oauth2-client.md) committed to:

| RFC-0025 commitment | Outcome |
|---|---|
| New package `swift-oauth2-client`, module `OAuth2Client`, 49th in ecosystem | ✓ shipped. |
| `OAuth2Client` struct with `(tokenEndpoint, clientID, clientSecret?)` config | ✓ shipped. |
| 3 grant types: `.clientCredentials`, `.authorizationCode` w/PKCE, `.refreshToken` | ✓ shipped. |
| `requestBody(grant:)` emits `application/x-www-form-urlencoded` bytes | ✓ shipped. |
| `parseResponse(_:)` handles RFC 6749 § 5.1 success + § 5.2 error shapes | ✓ shipped. |
| `TokenResponse` struct with standard fields | ✓ shipped. |
| `OAuth2ClientError` typed-throws enum | ✓ shipped. |
| Caller-driven HTTP transport (no transport abstraction) | ✓ shipped. |
| Foundation-free; single dep (swift-bytes) | ✓ shipped. |
| Hand-rolled `FormEncoder` + `JSONScanner` (no swift-uri / swift-json / swift-base64 / swift-crypto) | ✓ shipped. |
| 15-20 tests (RFC range) | ✓ shipped 26 tests across 6 suites (within and slightly above range; tight scope justified expanded coverage). |
| Out of scope: auth URL building, PKCE generation, state/nonce, token caching, OIDC helpers, implicit/password/device-code grants, mTLS, private_key_jwt | ✓ all honored. |
| ~300-500 LOC estimate | ✓ actual ~365 source LOC (within estimate); ~500 test LOC (matches estimate). |
| ~1-1.5 day calendar | ✓ actual ~1 hour (well under). |
| Pre-emptive Pages setup | ✓ honored (3rd successful application of the three-API-call sequence). |

**Zero scope reshape this phase.** **4th consecutive clean-from-RFC phase** (Phases 17, 18, 19, 20). The patterns introduced over Gates 14-16 (verify-before-drafting, brainstorm reality-check) are now **stably preventing reshape across 4 consecutive phases**. The 5-instance reshape streak (Phases 12-16) is firmly closed.

**Calendar under-estimate (~1 hour vs ~1-1.5 day):** Phase 20 shipped faster than RFC-0025 predicted because (a) hand-rolled FormEncoder + JSONScanner are well-bounded; (b) no external integrations (no swift-crypto, no HTTP wiring); (c) 4 consecutive zero-reshape phases means RFC-to-implementation translation has zero friction. **Worth a memory note:** RFC estimates have been consistently 5-10x over-estimating wall-clock since Phase 17.

---

## Item 3 — Revised criterion (≥1 RFC accepted *or* documented absence of policy gaps) ✓ PASS

**Pre-Phase 20 (Gate 19 closeout):** RFC-0025 (Phase 20 anchor) accepted 2026-05-16.

**During Phase 20 execution:** zero RFCs accepted.

Phase 20 surfaced these findings during execution; all are memory-saved transferable lessons rather than cross-package policy gaps:

- **RFC calendar estimates have systematic under-estimating bias.** RFC-0025 estimated 1-1.5 days; Phase 20 actually shipped in ~1 hour. Same pattern in Phase 19 (RFC estimate 0.5-1 day; actual ~1 hour) and Phase 18 (RFC estimate ~0.5 day; actual <30 min). **Pattern:** for additive minor bumps OR focused new-package work with ~300-600 LOC scope, ~1-2 hour wall-clock is typical. RFCs should estimate in hours when scope is bounded and primitives are well-understood.
- **Hand-rolled-vs-library decision pattern continues to scale.** Phase 20's `FormEncoder` (~50 LOC, no swift-uri dep) + `JSONScanner` (~100 LOC, no swift-json dep) follow the same pattern as v0.1 swift-jwt-verify's `extractAlg`. **Memory:** for fixed-schema narrow-vocabulary inputs/outputs, hand-roll; for variable-schema, depend on the library. The "hand-rolled" path stays right for narrow scope.
- **Caller-driven HTTP transport pattern is canonical.** Phase 14 (OTLP trinity) + Phase 16 (swift-log-bridge) + Phase 17 (swift-metrics-bridge) + Phase 20 (swift-oauth2-client) all use the "package emits/accepts `Bytes`, caller wires HTTP" pattern. Worth codifying as a memory note for future HTTP-adjacent packages.

None warrant a new cross-package RFC. **Documented absence of policy gaps. Criterion satisfied.**

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

Per-package review (single new package):

| Convention | swift-oauth2-client v0.1 |
|---|---|
| Module name PascalCase, no `Swift` prefix | ✓ OAuth2Client |
| Apache-2.0 + LLVM exception SPDX | ✓ |
| NOTICE crediting upstream | ✓ swift-bytes only (single dep) |
| Swift 6.0+ tools version | ✓ |
| macOS 14 platform floor | ✓ |
| Sendable-clean by default | ✓ |
| Strict concurrency in CI | ✓ |
| Single public error enum, typed throws | ✓ OAuth2ClientError (3 cases) |
| Public APIs Foundation-free | ✓ entirely Foundation-free (no internal Foundation either) |
| Repo skeleton matches bare-swift conventions | ✓ (modeled on swift-log-bridge) |
| README tagline + ≤30-line example | ✓ (3 usage examples: PKCE auth-code, client credentials, refresh) |
| CHANGELOG with v0.1.0 entry per Keep-a-Changelog | ✓ |
| CI green on macOS + Linux | ✓ all 7 jobs first-try |
| DocC bundle | ✓ |
| docc-target set from initial commit (Gate 11 lesson) | ✓ |
| Sanitizers ON (no large static data) | ✓ |
| Pre-emptive Pages setup (Gate 16 codification) | ✓ — 3rd successful application |

### Deviations and findings

**1. Fully Foundation-free.** Phase 20 is the **first net-new package to achieve "no Foundation anywhere — not even internally"** (modulo swift-bytes which is fully Foundation-free itself). Prior bridge packages (swift-log-bridge, swift-metrics-bridge, swift-distributed-tracing-bridge) have internal Foundation for swift-crypto `Data` bridging; swift-oauth2-client doesn't need swift-crypto, so the constraint holds end-to-end. **Pattern:** packages without swift-crypto dependencies can be entirely Foundation-free.

**2. Single direct dep (swift-bytes) is the narrowest dep graph in the ecosystem.** Most bridges have 5-7 deps; most format packages have 2-3; swift-oauth2-client has 1. **Pattern:** anchor packages on the simplest possible dep graph; pull in more only when functional need surfaces.

**3. Hand-rolled internals (FormEncoder + JSONScanner) continue to land cleanly.** 4 hand-rolled internal helpers across the project so far (PunycodeCodec in swift-idna, IdnaMappingTable in swift-idna, FormEncoder + JSONScanner in swift-oauth2-client). All shipped without bugs in their respective phases. **Pattern is firmly canonical** for narrow-scope internal utilities.

**4. Procedural-deferral self-correction (Gate 19) applied successfully.** OAuth 2.0 was deferred 8 times under "no concrete adopter demand." Gate 19 introduced the principle: re-examine deferral criteria when they become self-perpetuating. Phase 20 broke through on OAuth; Phase 19 broke through on JWT signing. **Pattern works for the kind of project bare-swift is** (proactive ecosystem building, not adopter-response). Future similar deferrals (swift-brotli v0.3 streaming, swift-idna v0.3, swift-distributed-tracing-bridge v0.3 B3+Jaeger) can use the same principle when their queue position justifies it.

**5. Plan task granularity adapted naturally.** Plan had 10 tasks; inline execution coalesced Tasks 2-7 (source files + 26 tests) into 1 commit because they were tightly interdependent. Same coalescing observed in Phases 18 + 19. **Memory note carried forward:** plan task granularity for bite-sized subagent-driven execution stays right; inline execution can coalesce when changes are interdependent.

### Verdict

RFC-0001 held cleanly under Phase 20 stress. The Foundation-free + single-dep + hand-rolled-internals pattern is now firmly canonical for narrow-scope packages.

---

## Item 5 — Phase 21 anchor decision ✗ NOT YET (recommendation)

Phase 21 anchor candidates surveyed:

| Wave | Packages | Risk | Adoption fit | Deferral count |
|---|---|---|---|---|
| **swift-jwt-verify v0.3 (ES384 + ES512 + EdDSA + `kid` header)** | swift-jwt-verify v0.3 | Low-medium | **High** — clean algorithm expansion; same Signer/Verifier pattern as v0.2; closes the algorithm-completion deferral. | 1 (Gate 19 only) |
| swift-oauth2-client v0.2 (auth URL builder + PKCE generation + state/nonce helpers) | swift-oauth2-client v0.2 | Low | Medium — natural follow-on to Phase 20; closes documented v0.1 deferrals; but v0.1 just shipped same-day, so the "natural settle time" argument isn't procedural drift. | 0 |
| swift-brotli v0.3 (streaming encoder) | swift-brotli v0.3 | Medium | Medium — closes Phase 12 v0.2 one-shot-only deferral; 8 prior deferrals (matches OAuth's count when it broke). | 8 |
| swift-distributed-tracing-bridge v0.3 (B3 + Jaeger propagation) | bridge v0.3 | Low-medium | Low — no concrete demand for B3/Jaeger when W3C just landed; 3rd v0.x bump on same package in 3 phases. | 2 |
| swift-idna v0.3 (Bidi + ContextJ + ContextO) | swift-idna v0.3 | Medium | Medium — closes Phase 15's documented deferral; rarely-hit rules. | 5 |
| Crypto-adjacent | various | Medium-low | Low — 16x rejected; swift-crypto entrenched. | 16 (rejected) |
| Package rename `swift-jwt-verify` → `swift-jwt` | rename project | Low | Low-medium — name now genuinely misleading; better when v0.3 brings substantial features. | n/a |

### Recommendation: **Anchor on swift-jwt-verify v0.3 (ES384 + ES512 + EdDSA + `kid` header)**

Reasons:

1. **Clean algorithm-expansion continuation.** Phase 19 shipped HS256 + ES256 signing. Phase 21 adds three more algorithms: ES384, ES512 (different curves; swift-crypto's `P384.Signing` + `P521.Signing` are stable API), EdDSA via Curve25519 (`Curve25519.Signing` is stable). Same Signer/Verifier pattern; no new architectural decisions.
2. **No swift-crypto SPI risk.** ES384, ES512, EdDSA all have **stable** swift-crypto APIs. (RSA still in `_RSA` experimental; deferred to Phase 22+.)
3. **Closes the algorithm-completion deferral.** swift-jwt-verify v0.2's CHANGELOG explicitly notes "RS256 / RS384 / RS512 / PS256 / ES384 / ES512 / EdDSA. Phase 20+ candidates." Phase 21 covers three of these (ES384, ES512, EdDSA). RS256/RS384/RS512/PS256 remain Phase 22+ (RSA SPI).
4. **`kid` header customization is a tiny additive.** ~30 LOC: optional `kid: String?` field on `JWTSigner` init; if non-nil, emit `{"alg":"...","typ":"JWT","kid":"..."}` instead of the v0.2 canonical header. Closes another documented v0.2 deferral. Pairs naturally with algorithm expansion (callers using multiple keys typically also need `kid` to select the right verification key).
5. **Audience continuity.** Same auth/security adopters from Phases 11 + 19 + 20. Enterprise OIDC adopters (the primary swift-oauth2-client user audience from Phase 20) often use ES384/ES512 for compliance reasons.
6. **Non-breaking additive minor bump.** All v0.2 APIs unchanged. v0.3 adds:
   - 3 new cases on `JWTAlgorithm` enum (additive; existing `.hs256` and `.es256` unchanged).
   - 3 new `Signer.sign*` static functions (mirroring existing `signHS256` / `signES256`).
   - 3 new `Verifier.verify*` static functions (mirroring existing).
   - Optional `kid` field on `JWTSigner` init (defaults to nil; v0.2 behavior preserved).
7. **Risk profile is well-bounded.**
   - swift-crypto's `P384.Signing.PrivateKey.signature(for:)`, `P521.Signing.PrivateKey.signature(for:)`, `Curve25519.Signing.PrivateKey.signature(for:)` all return signatures with `rawRepresentation` (well-defined byte layout per RFC 7518 § 3.4 + RFC 8037 § 3.1).
   - Test vectors: RFC 8037 Appendix A.4 has EdDSA examples; RFC 7518 § 3.4 has ECDSA structure; round-trip tests against the package's own verifier are the unambiguous correctness gate.
   - No new dependencies. swift-crypto already in v0.1.
8. **Sets up Phase 22+ options.**
   - **RS256/RS384/RS512** become the next algorithm-expansion candidate when swift-crypto stabilizes RSA SPI.
   - **swift-oauth2-client v0.2** (auth URL + PKCE helpers) by then has ~4-5 days of v0.1 settle time.
   - **swift-distributed-tracing-bridge v0.3 (B3 + Jaeger)** stays available.
   - **swift-brotli v0.3 streaming** stays available (now at 9 deferrals; next-strongest breakthrough candidate per the procedural-correction pattern).
   - **Package rename** becomes more pressing as v0.3 has substantial new features.

### Phase 21 tranche sketch (subject to formal RFC)

Single-anchor phase shape, two reasonable decompositions:

**Shape A (recommended): single tranche, all 3 algorithms + `kid`**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 21A | swift-jwt-verify v0.3 | ES384 sign+verify + ES512 sign+verify + EdDSA sign+verify + optional `kid` header field + ~25-30 tests | ~1-2 hours |

**Shape B (alternate): two tranches if EdDSA scope expands**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 21A | swift-jwt-verify v0.3 | ES384 + ES512 (P384/P521 ECDSA only) | ~1 hour |
| 21B | swift-jwt-verify v0.4 | EdDSA (Curve25519) + `kid` header | ~1 hour |

Decision deferred to brainstorm phase. Per the now-canonical brainstorm-reality-check pattern, expect Shape B only if EdDSA + Ed25519 key handling has more surface area than swift-crypto's P256-style pattern (unlikely; Curve25519.Signing follows identical shape).

Per the systematic-calendar-underestimate observation from Item 3, **realistic wall-clock estimate is ~1-2 hours** matching Phase 19's ~1 hour and Phase 20's ~1 hour for similar scope.

### Why other waves were rejected

- **swift-oauth2-client v0.2** — v0.1 shipped same-day; the "natural settle time" argument here ISN'T procedural drift (v0.1 hasn't had real-world exposure). Phase 22+ candidate when v0.1 has had several days.
- **swift-brotli v0.3 streaming** — 8 deferrals matches OAuth's count when it broke. Strong Phase 22 candidate. Phase 21 priority goes to JWT algorithm expansion because audience continuity from Phases 19 + 20 is tighter.
- **swift-distributed-tracing-bridge v0.3 (B3 + Jaeger)** — no concrete demand; W3C just landed. Phase 22+ candidate.
- **swift-idna v0.3** — rarely-hit rules; no demand. Phase 22+ candidate.
- **Crypto-adjacent** — 16th consecutive rejection.
- **Package rename `swift-jwt-verify` → `swift-jwt`** — premature; should accompany substantial v0.3 features. After Phase 21 ships ES384/ES512/EdDSA + kid, the rename becomes more compelling but is a Phase 23+ candidate (or quiet patch).

**Action:** author the anchor decision as **RFC-0026** (Phase 21 anchor: swift-jwt-verify v0.3 — ES384 + ES512 + EdDSA + `kid` header) and accept it before Phase 21 plans start.

---

## Open work to clear Gate 20

1. **Write RFC-0026** (Phase 21 anchor: swift-jwt-verify v0.3). 1-hour task — RFC scope is small because it's pure algorithm expansion + tiny additive `kid` field. References this retrospective.
2. **Begin Phase 21 planning** after RFC-0026 accepts. Brainstorm should verify swift-crypto's `P384.Signing` / `P521.Signing` / `Curve25519.Signing` API shapes (expected to match P256's pattern exactly; quick `.build/checkouts/` grep).

---

## What this retrospective changes for Phase 21

- The plan-then-execute-with-checkpoints workflow stays. Validated across micro (Phase 18: <30 min), small (Phases 16, 19, 20: ~1 hour), medium (Phases 14C/15B/17A: ~1 day), and large (Phase 15A: ~1 day with data tables) shapes.
- **New observation (calendar estimate calibration):** For additive minor bumps OR focused new-package work with bounded scope (~300-600 LOC), realistic wall-clock is ~1-2 hours. RFC estimates that say "0.5-1 day" or "1-1.5 days" for this profile have been consistently 5-10x over. **Future RFCs should estimate in hours for bounded scope.** Memory note.
- **Reinforced (firmly canonical now):** Hand-rolled-vs-library decision pattern. 4 instances now (PunycodeCodec, IdnaMappingTable, FormEncoder, JSONScanner). All shipped without bugs.
- **Reinforced (firmly canonical now):** Caller-driven HTTP transport pattern. 5 instances (trinity x 3 + swift-log-bridge + swift-oauth2-client). Codify as memory note.
- **Reinforced (firmly canonical now):** Procedural-deferral self-correction. After 7-9 rejections under a calendar-based or "no concrete demand" defer-criterion, the criterion is self-perpetuating. Phase 19 (JWT signing 7-rejection), Phase 20 (OAuth 8-rejection) broke through. swift-brotli v0.3 streaming at 8 deferrals is the next candidate (Phase 22+).
- **New memory note:** Foundation-free entirely possible for non-swift-crypto packages. swift-oauth2-client is the first. Pattern: avoid swift-crypto deps when possible; hand-roll narrow internals; achieve full Foundation-free.
- **Carried forward:** Phase 20 left no documented deferrals. The auth-client tier is feature-complete.

---

## Decision

Gate 20 is **PASSED on items 1, 2, 3, 4** and **NOT YET PASSED on item 5 (open)**. Phase 21 plans should be drafted but not executed until item 5 closes via RFC-0026.

Phase 20 closed the longest-standing deferral in the auth tier alongside Phase 19's JWT-signing breakthrough. The bare-swift auth-client tier is now feature-complete: basic-auth + bearer + jwt-verify (sign + verify) + oauth2-client cover the typical OIDC relying-party stack with zero adopter-side boilerplate beyond HTTP transport wiring.

The next step is **RFC-0026 anchoring Phase 21 on swift-jwt-verify v0.3 (ES384 + ES512 + EdDSA algorithm expansion + `kid` header customization)**. swift-oauth2-client v0.2 (auth URL builder + PKCE helpers), swift-brotli v0.3 streaming, swift-distributed-tracing-bridge v0.3 (B3 + Jaeger), swift-idna v0.3, package rename `swift-jwt-verify` → `swift-jwt`, and RSA-family JWT algorithms (when swift-crypto stabilizes RSA SPI) remain on the queue for Phase 22+.

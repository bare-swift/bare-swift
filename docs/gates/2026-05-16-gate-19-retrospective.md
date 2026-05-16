# Gate 19 Retrospective: Phase 19 → Phase 20

**Date:** 2026-05-16
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 19 (the swift-jwt-verify signing wave anchored by [RFC-0024](../../rfcs/0024-phase-19-anchor-swift-jwt-verify-v0.2-signing.md)) against the Gate 1–18 criteria template and recommends whether Phase 20 should start. Phase 19 was a **single-tranche additive minor bump** (19A swift-jwt-verify v0.2). Shipped cleanly.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All anchored Tranche 19A deliverables shipped, CI green, DocC live, listed | ✓ PASS |
| 2 | RFC-0024 commitments fulfilled | ✓ PASS *(near-zero scope reshape; 3rd consecutive clean RFC since Phase 17)* |
| 3 | ≥1 substantive RFC accepted *or documented absence of policy gaps* | ✓ PASS (documented absence of policy gaps) |
| 4 | RFC-0001 conventions stress-test against Phase 19 deliverables | ✓ DONE BELOW |
| 5 | Phase 20 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0025 below) |

**Overall:** Items 1–4 pass. Item 5 is open. Phase 20 plans should be drafted but not executed until item 5 closes via RFC-0025.

The roadmap's stop conditions did not trigger: Phase 19 calendar time was ~1 hour wall-clock. No security incidents.

---

## Item 1 — Phase 19 deliverables shipped ✓ PASS

| Tranche | Package | Tag | New code | DocC | Index | CI |
|---|---|---|---|---|---|---|
| 19A | swift-jwt-verify | **v0.2.0** (existing-package minor bump) | ~140 source LOC + ~190 test LOC | ✓ | ✓ | green (first-try, push + tag) |

Single tranche shipped. **Closes the 7-rejection auth-tier verify-only deferral** — from Phase 11C (2026-05-12 v0.1 verifier) through Gates 11/12/13/14/15/16/17/18, 7 consecutive gate retros deferred signing under the "wait a week" criterion. By Phase 19 start (2026-05-16), v0.1 had 4 days of settle time with zero reported issues; ship was clean.

The auth tier now offers **full JWT primitive coverage** (sign + verify, HS256 + ES256) alongside swift-basic-auth + swift-bearer. Adopters who issue JWTs (authorization servers, identity providers, internal token mints) have a clean primitive instead of dropping to swift-crypto.

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists **48 packages** — no count change (existing-package minor bump).

**Calendar time:** ~1 hour wall-clock — Phase 18 (<30 min) is still the floor; Phase 19 is the second-smallest. The pattern of "small additive bump on existing primitive package" reliably ships in <1 hour.

**4-consecutive first-try-clean CI streak:** Phase 16 (swift-log-bridge), Phase 17 (swift-metrics-bridge), Phase 18 (swift-distributed-tracing-bridge v0.2), Phase 19 (swift-jwt-verify v0.2). Four phases in a row with zero CI fixup commits across push + tag + Release + Publish docs workflows.

---

## Item 2 — RFC-0024 commitments ✓ PASS (near-zero reshape)

[RFC-0024](../../rfcs/0024-phase-19-anchor-swift-jwt-verify-v0.2-signing.md) committed to:

| RFC-0024 commitment | Outcome |
|---|---|
| swift-jwt-verify v0.2 (existing package, additive minor bump) | ✓ shipped. |
| Public `JWTSigner` struct mirroring `JWTVerifier`'s shape | ✓ shipped. |
| Internal `Signer` namespace (mirror of `Verifier`) | ✓ shipped. |
| HS256 signing via `HMAC<SHA256>` | ✓ shipped. |
| ES256 signing via `P256.Signing.PrivateKey` (32-byte raw scalar) | ✓ shipped. |
| `Base64URL.encode(_:)` addition | ✓ shipped. |
| Canonical header byte constants (alg-first) | ✓ shipped. |
| Non-breaking — v0.1 verify APIs unchanged | ✓ honored. |
| No new dependencies | ✓ honored. |
| No new error cases — reuse `JWTVerifyError.invalidKey` | ✓ honored. |
| 12-15 tests (RFC range) | ✓ shipped 13 tests in single `SignerTests` suite (within range). |
| Out of scope: RS256, ES384, ES512, EdDSA, header customization, claim builder, JWE/JWK/JWKS, package rename | ✓ all honored. |

**Near-zero scope reshape this phase.** **3rd consecutive clean-from-RFC phase** (Phases 17, 18, 19). One micro-reshape: test #4's empty-payload round-trip changed to `{}` (minimal valid JSON) after discovering v0.1's verifier rejects tokens with empty payload segments. The signer itself emits the empty-payload token correctly; it just doesn't survive v0.1's strict verify check. Memory note added; v0.3+ may relax the verifier check if adopter demand surfaces.

**The "wait a week" rule was procedural drift.** Phase 19 broke through after 7 deferrals. Memory note for future similar deferrals: a self-imposed clock criterion stops serving when the rest of the project moves faster than the clock measures. The right question is "are there outstanding issues with v0.1 that should be addressed before extending the package?" — not "has enough calendar time elapsed?"

---

## Item 3 — Revised criterion (≥1 RFC accepted *or* documented absence of policy gaps) ✓ PASS

**Pre-Phase 19 (Gate 18 closeout):** RFC-0024 (Phase 19 anchor) accepted 2026-05-16.

**During Phase 19 execution:** zero RFCs accepted.

Phase 19 surfaced these findings during execution; all are memory-saved transferable lessons rather than cross-package policy gaps:

- **v0.1 JWT verifier rejects empty payload segments.** Test #4's `JWTSigner.sign([])` round-trip failed `JWTVerifier.verify` with `.malformedToken`. The signer correctly emits `<header>..<sig>` for empty payloads, but `JWTVerify.split` checks `second == first + 1` and rejects. Updated test to use `{}` (minimum valid JSON object). **Lesson:** when adding a signing side to an existing verify-side package, audit the verifier's input constraints — they implicitly constrain what the signer can produce that round-trips.
- **ES256 key bytes asymmetry codified.** Sign uses 32-byte raw scalar (`P256.Signing.PrivateKey.rawRepresentation`); verify uses 65-byte x963 uncompressed (`P256.Signing.PrivateKey.publicKey.x963Representation`). Both derived from a single `P256.Signing.PrivateKey()` construction. Documented in README + DocC + CHANGELOG explicitly.
- **Procedural deferral can be self-perpetuating.** As described in Item 2 — the "wait a week" rule was used 7 times to defer JWT signing while the project moved through 7 phases in 4 days. The actual gating criterion should be "are there outstanding issues?" not calendar time.

None warrant a new cross-package RFC. **Documented absence of policy gaps. Criterion satisfied.**

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

Per-package review (single existing-package minor bump):

| Convention | swift-jwt-verify v0.2 |
|---|---|
| Module name PascalCase, no `Swift` prefix | ✓ JWTVerify (unchanged; despite gaining sign capability) |
| Apache-2.0 + LLVM exception SPDX | ✓ |
| NOTICE crediting upstream | ✓ (unchanged from v0.1; swift-crypto + swift-base64) |
| Swift 6.0+ tools version | ✓ |
| macOS 14 platform floor | ✓ (unchanged) |
| Sendable-clean by default | ✓ |
| Strict concurrency in CI | ✓ |
| Single public error enum, typed throws | ✓ JWTVerifyError (unchanged; reused `.invalidKey`) |
| Public APIs Foundation-free | ✓ (Foundation only in internal `Signer.swift` for swift-crypto `Data` bridging — matches v0.1's `Verifier.swift`) |
| README tagline + ≤30-line example | ✓ updated with HS256/ES256 sign+verify round-trips |
| CHANGELOG with v0.2.0 entry per Keep-a-Changelog | ✓ |
| CI green on macOS + Linux | ✓ all 7 jobs first-try |
| DocC bundle | ✓ updated with Signing topic |
| Sanitizers ON (no large static data) | ✓ |
| Non-breaking minor bump pattern | ✓ (additive; no v0.1 API touched) |

### Deviations and findings

**1. Module name `JWTVerify` despite gaining sign capability.** Phase 19 deliberately didn't rename. The package name (`swift-jwt-verify`) is technically misleading after v0.2 — it does both. Rename to `swift-jwt` deferred to Phase 20+ when adopter feedback surfaces. The README + DocC + CHANGELOG explicitly note this naming choice. Pattern: don't churn names mid-phase-cycle; let v0.x stabilize on multiple algorithm fronts first, then rename when scope justifies the migration cost.

**2. Foundation-in-internal-only pattern consistent.** v0.2's `Signer.swift` imports Foundation for `Data(signingInput)` — matches v0.1's `Verifier.swift` convention. Both files have explicit comments noting "internal use only — bridge to swift-crypto's Data API". Public API of JWTVerify stays Foundation-free. Pattern is canonical for swift-crypto wrapping packages.

**3. ES256 key bytes asymmetry deserves first-class documentation.** This is unusual — most APIs use symmetric or identically-formatted key bytes. The 32-byte private vs 65-byte public split confused me momentarily during the brainstorm; documented it three times (CHANGELOG, README, DocC, struct doc comment) to make the conversion idiom (`priv.rawRepresentation` + `priv.publicKey.x963Representation`) the discoverable canonical pattern.

**4. RFC v0.x-bump pattern continues to scale across packages.** Phase 18 (swift-distributed-tracing-bridge v0.1→v0.2) and Phase 19 (swift-jwt-verify v0.1→v0.2) both used the same shape: dep-bump-or-not + ~30-150 source LOC + focused test suite + docs + ship. ~30-60 min wall-clock each. Pattern is canonical for additive minor bumps on existing packages.

**5. Plan execution naturally coalescing.** Plan had 8 tasks; inline execution combined Tasks 2-4 (Signer + JWTSigner + tests) into one logical commit because the changes were tight and interdependent. Same coalescing as Phase 18. **Memory:** for small additive bumps, the plan's bite-sized task granularity informs structure but doesn't dictate commit boundaries.

### Verdict

RFC-0001 held cleanly under Phase 19 stress. The non-breaking-minor-bump pattern continues to ship reliably in <1 hour.

---

## Item 5 — Phase 20 anchor decision ✗ NOT YET (recommendation)

Phase 20 anchor candidates surveyed:

| Wave | Packages | Risk | Adoption fit |
|---|---|---|---|
| **OAuth 2.0 client primitives (new package)** | swift-oauth2-client v0.1 | Medium | **High** — 9th rejection threshold reached on OAuth 2.0 (Gates 11-19 all deferred); composes on top of newly-completed JWT sign+verify; natural next layer in the auth tier after Phase 19. |
| swift-jwt-verify v0.3 (RS256) | swift-jwt-verify v0.3 | Medium-high | Medium — RS256 dominates JWT in OIDC/enterprise; swift-crypto's RSA API is still under `_RSA` (experimental SPI), making adopter-facing exposure risky. |
| swift-jwt-verify v0.3 (ES384 + ES512 + EdDSA) | swift-jwt-verify v0.3 | Medium | Medium — clean algorithm expansion; same Signer/Verifier pattern as v0.2; ~250-400 LOC. |
| swift-distributed-tracing-bridge v0.3 (B3 + Jaeger propagation) | bridge v0.3 | Low-medium | Low — no concrete adopter demand for B3/Jaeger when W3C just landed. |
| swift-idna v0.3 (Bidi + ContextJ + ContextO) | swift-idna v0.3 | Medium | Medium — closes Phase 15's documented deferral. |
| swift-brotli v0.3 (streaming encoder) | swift-brotli v0.3 | Medium | Medium — closes Phase 12 v0.2 one-shot-only deferral; 7 prior deferrals. |
| Header customization in swift-jwt-verify (kid, jku, x5u, cty) | swift-jwt-verify v0.3 | Low | Medium — small, but bundles best with RS256 or ES384/ES512/EdDSA. |
| Crypto-adjacent | various | Medium-low | Low — 16th rejection candidate. |
| swift-publicsuffix v0.2 (ICANN/PRIVATE split) | swift-publicsuffix v0.2 | Low | Low — narrow; quiet patch when adopter asks. |

### Recommendation: **Anchor on OAuth 2.0 client primitives (new package)**

Reasons:

1. **9th rejection threshold reached on OAuth 2.0.** OAuth 2.0 has been deferred at every gate since Phase 11 (Gates 11-18, 8 prior deferrals). Phase 19 used the same "8th-rejection threshold" reasoning to break through on JWT signing. Same logic applies to OAuth: "no concrete adopter demand" has been the canonical defer-reason; but everything in bare-swift is built proactively, and the auth tier is now feature-complete on its primitives (basic-auth + bearer + jwt sign+verify) — OAuth is the natural composition layer.
2. **Composes on the just-completed JWT primitive.** Phase 19's `JWTSigner` makes OAuth 2.0 issuer flows (e.g., id-token signing) practical. Phase 11C's `JWTVerifier` makes OAuth 2.0 relying-party flows (e.g., id-token verification) practical. OAuth 2.0 client is the layer that ties them together via the token endpoint.
3. **Net-new package keeps SemVer churn manageable.** Three consecutive existing-package minor bumps (Phase 17 swift-metrics-bridge, Phase 18 swift-distributed-tracing-bridge v0.2, Phase 19 swift-jwt-verify v0.2). Phase 20 net-new gives adopters' SemVer surface a rest.
4. **Audience continuity.** Same auth/security adopters from Phases 11 (basic-auth + bearer + jwt-verify) and 19 (jwt-verify signing).
5. **Risk profile is well-bounded if v0.1 is honest-scoped.**
   - Token request only: `client_credentials` + `authorization_code` (with PKCE) + `refresh_token` grants.
   - HTTP transport: caller wires it (matches the trinity's caller-driven-transport pattern).
   - JSON parsing: token response parsing. Either hand-roll (~30 LOC for the 4 known fields) or depend on swift-json (Phase 6+; in-house).
   - PKCE generation: caller does it (PKCE is just `SHA256(verifier).base64url` — easy with swift-crypto). v0.1 just accepts the pre-computed verifier + challenge.
   - State + nonce: caller does it (random bytes — easy with swift-crypto).
   - Authorization URL: caller builds it (URL is a string template per RFC 6749 § 4.1.1; swift-uri is available for adopters who want a typed builder).
   - **Honest v0.1 scope: token request operation only.** ~300-500 LOC.
6. **Sets up Phase 21+ options.**
   - **swift-jwt-verify v0.3 (RS256 OR ES384/ES512/EdDSA)** — natural follow-on once swift-crypto's RSA SPI stabilizes (RS256) or as a clean ECDSA/EdDSA expansion.
   - **swift-distributed-tracing-bridge v0.3 (B3 + Jaeger)** stays available.
   - **swift-idna v0.3 / swift-brotli v0.3 streaming** stay available.
   - **Package rename swift-jwt-verify → swift-jwt** becomes more pressing as adopters write OAuth code referencing both `JWTVerifier` and `JWTSigner` from a package named "verify".

### Phase 20 tranche sketch (subject to formal RFC)

**Shape A (recommended): single tranche, OAuth 2.0 client v0.1**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 20A | swift-oauth2-client v0.1 | `OAuth2Client` struct + `requestToken(grant:)` + 3 `GrantType` cases (clientCredentials, authorizationCode w/PKCE, refreshToken) + response parsing + error enum + ~15-20 tests | ~1-1.5 days |

**Shape B (alternate): two tranches if HTTP-transport abstraction expands**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 20A | swift-oauth2-client v0.1 | Token request + parsing only; caller wires bytes-to-HTTP-transport via the same closure pattern as OTLPMetricsFactory's `flushExport` | ~1 day |
| 20B | swift-oauth2-client v0.2 | Helpers for authorization URL building + state/nonce generation | ~0.5 day |

Per the now-canonical brainstorm-phase reality-check pattern, expect Shape B if `requestToken` needs an opinionated HTTP transport abstraction (it might — adopters need to send `application/x-www-form-urlencoded` POST bodies and parse `application/json` responses).

### Why other waves were rejected

- **swift-jwt-verify v0.3 (RS256)** — swift-crypto's RSA API is still in `_RSA` (experimental SPI). Exposing it to adopters before stabilization risks v0.3 → v0.4 breaking churn when swift-crypto promotes the API. Defer to Phase 22+ when swift-crypto stabilizes.
- **swift-jwt-verify v0.3 (ES384 + ES512 + EdDSA)** — third v0.x bump on same package in three phases would be aggressive. Phase 21+ candidate after OAuth 2.0 ships.
- **swift-distributed-tracing-bridge v0.3 (B3 + Jaeger)** — no concrete adopter demand. Phase 21+ candidate.
- **swift-idna v0.3** — rarely-hit rules; no demand. Phase 21+ candidate.
- **swift-brotli v0.3 streaming** — narrower than OAuth 2.0 closure of auth tier. Phase 21+ candidate (8th gate-deferral; equal-priority with OAuth in absolute deferral count but lower in audience fit).
- **OAuth 2.0 + crypto-adjacent bundle** — 16th rejection of crypto-adjacent. swift-crypto remains entrenched.
- **swift-publicsuffix v0.2** — narrow; quiet patch.
- **Package rename swift-jwt-verify → swift-jwt** — premature; let v0.2 settle first; rename when v0.3 features justify the migration cost.

**Action:** author the anchor decision as **RFC-0025** (Phase 20 anchor: swift-oauth2-client v0.1) and accept it before Phase 20 plans start.

---

## Open work to clear Gate 19

1. **Write RFC-0025** (Phase 20 anchor: swift-oauth2-client v0.1). 1-2 hour task. References this retrospective.
2. **Begin Phase 20 planning** after RFC-0025 accepts. Brainstorm should verify the OAuth 2.0 RFC 6749 token-endpoint shape against any in-house reference (none yet) and decide on the HTTP transport abstraction (caller closure vs typed adapter).

---

## What this retrospective changes for Phase 20

- The plan-then-execute-with-checkpoints workflow stays. Validated across micro (Phase 18: <30 min), small (Phases 16, 19: ~1 hour), medium (Phases 14C/15B/17A: ~1 day), and large (Phase 15A: ~1 day with data tables) shapes.
- **Reinforced (firmly canonical now):** Procedural-deferral self-correction. After 7-9 rejections under a calendar-based defer-criterion, re-examine whether the criterion is actually serving the project or just self-perpetuating. Phase 19 broke through on JWT signing; Phase 20 will break through on OAuth 2.0.
- **Reinforced (firmly canonical now):** Foundation-in-internal-only pattern for swift-crypto wrapping packages. Public API stays Foundation-free; internal source files that bridge to `Data` get an explicit comment noting "internal use only — bridge to swift-crypto's Data API."
- **New observation:** Plan task granularity should adapt to execution mode. 8-task plan ran as 4 inline commits because logically-adjacent tasks coalesced naturally. Future plans can keep bite-sized granularity for subagent-driven execution; inline can coalesce.
- **New memory note:** When adding a signer-side to a verify-only package, audit the verifier's input constraints — they implicitly constrain what the signer can produce that round-trips. (v0.1's `JWTVerify.split` rejects empty payload segments; v0.2 signer's empty-payload test needed adjustment.)
- **Carried forward:** Phase 19 left no documented deferrals. The auth tier's JWT primitive is feature-complete on HS256 + ES256 sign+verify.

---

## Decision

Gate 19 is **PASSED on items 1, 2, 3, 4** and **NOT YET PASSED on item 5 (open)**. Phase 20 plans should be drafted but not executed until item 5 closes via RFC-0025.

Phase 19 closed the longest-standing functional gap in the auth tier — the verify-only design boundary deferred for 7 consecutive gates under a "wait a week" criterion that had stopped serving the project. The auth tier's JWT primitive is now feature-complete; adopters who issue tokens (authorization servers, identity providers) and adopters who consume tokens (resource servers, OAuth clients) both have clean primitives.

The next step is **RFC-0025 anchoring Phase 20 on swift-oauth2-client v0.1** (new package, 49th in ecosystem), which closes the 8-rejection deferral on OAuth 2.0 and adds the composition layer that ties swift-jwt-verify + swift-bearer + swift-basic-auth into a complete auth-client stack. swift-jwt-verify v0.3 (RS256 or ES384/ES512/EdDSA), swift-distributed-tracing-bridge v0.3 (B3 + Jaeger), swift-idna v0.3, swift-brotli v0.3 streaming, and a possible `swift-jwt` package rename remain on the queue for Phase 21+.

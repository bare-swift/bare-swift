# Gate 26 Retrospective: Phase 26 → Phase 27

**Date:** 2026-05-16
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 26 (the swift-oauth2-client v0.2 auth-flow-start wave anchored by [RFC-0031](../../rfcs/0031-phase-26-anchor-swift-oauth2-client-v0.2-auth-flow.md)) against the Gate 1–25 criteria template and recommends whether Phase 27 should start. Phase 26 was a **single-tranche existing-package minor bump** (26A swift-oauth2-client v0.2). Shipped cleanly.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All anchored Tranche 26A deliverables shipped, CI green, DocC live, index bumped | ✓ PASS |
| 2 | RFC-0031 commitments fulfilled | ✓ PASS *(zero scope reshape; 10th consecutive clean RFC since Phase 17)* |
| 3 | ≥1 substantive RFC accepted *or documented absence of policy gaps* | ✓ PASS |
| 4 | RFC-0001 conventions stress-test against Phase 26 deliverables | ✓ DONE BELOW |
| 5 | Phase 27 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0032 below) |

**Overall:** Items 1–4 pass. Item 5 is open. Phase 27 plans should be drafted but not executed until item 5 closes via RFC-0032.

Phase 26 calendar time was ~1.5 hours wall-clock (within RFC-0031's 1-2 hour bracket for net-new-features packages).

---

## Item 1 — Phase 26 deliverables shipped ✓ PASS

| Tranche | Package | Tag | New code | DocC | Index | CI |
|---|---|---|---|---|---|---|
| 26A | swift-oauth2-client | **v0.2.0** | ~440 source LOC + ~440 test LOC + ~60 docs LOC | ✓ | ✓ (0.1.0→0.2.0) | green (first-try) |

Single tranche shipped. **Completes OAuth 2.0 client coverage** — v0.1 covered the token endpoint (back half), v0.2 covers the auth flow start (front half).

**Phase 26 final tally:**
- 4 new public APIs: `OAuth2Client.authorizationURL(...)`, `OAuth2Client.PKCE` namespace, `OAuth2Client.randomToken(byteCount:)`, `ClientAuthMethod` enum + `OAuth2Client.basicAuthHeader()`.
- 4 new source files (Base64.swift, ClientAuthMethod.swift, PKCE.swift, extended OAuth2Client.swift).
- 4 new test files (Base64Tests, PKCETests, AuthorizationURLTests, ClientAuthMethodTests).
- 52 tests across 10 suites (26 v0.1 + 26 new); first-try CI green.
- swift-crypto 3.0+ dep added (first time on this package).
- ~1.5 hours wall-clock — within RFC-0031's "1-2 hours" estimate.
- **12-consecutive first-try-clean CI streak** (Phases 16-26).
- **10-consecutive zero-scope-reshape RFC streak** (Phases 17-26).
- v0.1 byte-for-byte preserved for `.body` auth method (regression-tested via dedicated test).

**Auth-tier story FEATURE-COMPLETE:**
- swift-basic-auth (Phase 11A v0.1) — HTTP Basic credential parsing
- swift-bearer (Phase 11B v0.1) — RFC 6750 Bearer + WWW-Authenticate
- swift-jwt-verify (Phase 11C v0.1 + 19A v0.2 + 21A v0.3) — JWT sign + verify (HS256/ES256/ES384/ES512/EdDSA + kid)
- swift-oauth2-client (Phase 20A v0.1 + **Phase 26A v0.2**) — token endpoint + auth flow start

Full OIDC relying-party stack now requires zero adopter-side boilerplate beyond HTTP transport wiring.

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists **49 packages** — no count change.

---

## Item 2 — RFC-0031 commitments ✓ PASS (zero scope reshape)

[RFC-0031](../../rfcs/0031-phase-26-anchor-swift-oauth2-client-v0.2-auth-flow.md) committed to:

| RFC-0031 commitment | Outcome |
|---|---|
| Existing package swift-oauth2-client v0.1 → v0.2 (additive minor bump) | ✓ shipped. |
| Authorization URL builder (RFC 6749 § 4.1.1) | ✓ shipped with all spec'd params (scope/state/nonce/code_challenge/responseType/additionalParams). |
| PKCE verifier/challenge primitives (RFC 7636) | ✓ shipped. RFC 7636 § 4.2 worked-example test passes. |
| `randomToken(byteCount:)` for state/nonce | ✓ shipped. |
| `ClientAuthMethod` enum + `clientAuthMethod` field + `basicAuthHeader()` | ✓ shipped with RFC 6749 § 2.3.1 form-urlencode-before-base64 semantics. |
| swift-crypto dep added | ✓ shipped (3.0+). |
| swift-base64 dep avoided (inline Base64 helper) | ✓ inline ~80 LOC standard + URL-safe variants. |
| Calendar estimate **deferred to brainstorm reality-check** | ✓ honored. Reality-check confirmed swift-crypto SHA256 + stdlib CSPRNG + FormEncoder reuse upfront. Estimate locked at 1.5-2 hours; actual ~1.5 hours. |
| Non-breaking on v0.1 | ✓ confirmed. v0.1 byte-equality preserved for `.body` auth method (regression test passes). |
| ~25-30 tests (RFC range) | ✓ shipped 26 new tests (mid-bracket). |
| Out of scope: token caching, OIDC helpers, implicit/password/device-code grants, mTLS, private_key_jwt, HTTP transport | ✓ all honored. |

**Zero scope reshape this phase.** **10th consecutive clean-from-RFC phase** (Phases 17-26).

**Estimate-vs-actual:** RFC-0031 said "1-2 hours pending reality-check"; actual ~1.5 hours. Within bracket. For net-new-features packages (Phase 26 was not a wrapper), the 1-2 hour estimate fits — distinct from the 30-60 min wrapper-package calibration.

---

## Item 3 — Revised criterion ✓ PASS

**Pre-Phase 26 (Gate 25 closeout):** RFC-0031 accepted 2026-05-16.

**During Phase 26 execution:** zero RFCs accepted.

Phase 26 surfaced findings worth codifying:

- **swift-crypto dep is canonical when SHA256/HMAC/random is needed.** 6 packages in the ecosystem now depend on swift-crypto (jwt-verify, log-bridge, metrics-bridge, distributed-tracing-bridge, jwt-verify v0.3 algo expansion, oauth2-client v0.2). When a package needs cryptographic primitives, add swift-crypto. Foundation-in-internal-only pattern keeps public API Foundation-free.
- **Stdlib's `SystemRandomNumberGenerator` is sufficient for CSPRNG needs** (OAuth state/nonce/PKCE-verifier). No need to pull in swift-crypto's random APIs — stdlib's underlying CSPRNG (arc4random on macOS, getentropy on Linux) is well-specified. **Memory:** when a package needs only secure random (no hashing/signing), stay swift-crypto-free.
- **Inline base64 helper continues the hand-rolled-narrow-vocabulary pattern.** ~80 LOC supports standard + URL-safe variants; matches v0.1's FormEncoder + JSONScanner pattern.
- **RFC 6749 § 2.3.1 form-urlencode-before-base64 detail.** Worth flagging: OAuth Basic auth requires URL-encoding `client_id` and `client_secret` BEFORE base64-encoding the `id:secret` pair. Different from plain RFC 7617 Basic. Future OAuth packages should heed this subtle requirement.
- **`@inlinable` references private statics** is a Swift constraint. Hit at Base64.swift during execution; SourceKit flagged it. Resolved by removing `@inlinable` (the perf benefit is marginal for an OAuth client). **Memory:** when adding `@inlinable` to an internal helper, ensure all referenced types/statics are `@usableFromInline internal` or wider, NOT `private`.
- **Reality-check before RFC estimate at 5/5 successful applications.** Phase 26 reality-check confirmed all four design questions upfront (SHA256 API shape, CSPRNG choice, FormEncoder reuse, base64 inline strategy). Pattern is procedural law.

None warrant a new cross-package RFC.

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

| Convention | swift-oauth2-client v0.2 |
|---|---|
| Module name PascalCase | ✓ OAuth2Client |
| Apache-2.0 + LLVM exception SPDX | ✓ |
| Sendable-clean by default | ✓ all new types are Sendable |
| Single public error enum | ✓ OAuth2ClientError (3 cases; no new ones in v0.2) |
| Public APIs Foundation-free | ✓ |
| README tagline + ≤30-line example | ✓ updated with auth flow examples |
| CHANGELOG with v0.2.0 entry | ✓ |
| CI green on macOS + Linux | ✓ first-try |
| DocC bundle | ✓ |
| Sanitizers ON | ✓ |
| Existing-encoder-preserved-verbatim | ✓ v0.1 default behavior preserved by construction |

### Deviations and findings

**1. First swift-crypto dep on a package.** Phase 26 codified the path: when SHA256 is needed, add swift-crypto; Foundation-in-internal-only pattern keeps public API Foundation-free. **Pattern is now firmly canonical** for crypto-needing packages.

**2. No new `OAuth2ClientError` cases.** Phase 26 considered adding `.invalidAuthorizationURL` or `.invalidPKCEMethod` but concluded these are caller bugs (impossible inputs would fatalError naturally), not runtime errors. **Memory:** "absence of new error cases" is a positive signal — over-typed errors are a code smell.

**3. Reality-check pre-locked all 4 design decisions** before any code was written. Pattern works: SHA256 confirmed easy (`SHA256.hash(data:)`); CSPRNG via stdlib (`UInt8.random(in:using:)`); FormEncoder reuse confirmed (form-urlencoded matches query string per RFC 6749 § 3.1); Base64 inline chosen over swift-base64 dep.

**4. RFC 7636 § 4.2 worked-example test** is a strong correctness gate. Future RFC-compliance work should include worked-example tests where the spec provides them.

### Verdict

RFC-0001 held cleanly under Phase 26 stress. Two new memory notes codified (CSPRNG-via-stdlib for random-only needs; `@inlinable` + private statics constraint).

---

## Item 5 — Phase 27 anchor decision ✗ NOT YET (recommendation)

Phase 27 anchor candidates surveyed:

| Wave | Packages | Risk | Adoption fit | Deferral count |
|---|---|---|---|---|
| **Codec-tier v0.4 drain() API sweep** (brotli + deflate + gzip + zlib) | 4 packages, 4 tranches | Medium (4-package coordinated) | **Medium-high** — closes documented Phase 25 multi-coding streaming deferral; completes streaming HTTP body story; each tranche is small (~30-60 min). | 2 each (Phases 25, 26) |
| Streaming decoders sweep (brotli + deflate + gzip + zlib v0.4 inflate) | 4 packages | High (3-5hr per tranche; major refactor of Decoder.swift / Inflater.swift state machines) | High — covers large-response-body decode use cases. | 1 each (just deferred from Phase 25) |
| Single-codec streaming decode (e.g., swift-deflate v0.4 inflate only) | 1 package | Medium-high (3-5hr) | Medium — starts decode sweep but doesn't complete it. | 0 |
| swift-oauth2-client v0.3 (token caching / refresh / OIDC helpers) | swift-oauth2-client v0.3 | Medium | High — but v0.2 has zero settle time (just shipped). | 0 (no deferral count yet) |
| swift-brotli v0.4 + swift-deflate v0.4 window carry | brotli v0.4, deflate v0.4 | Medium | Low-medium — ratio improvements. | 2 each (Phases 25, 26) |
| swift-distributed-tracing-bridge v0.3 (B3 + Jaeger) | bridge v0.3 | Low-medium | Low — no concrete demand; 7 deferrals. | 7 |
| swift-idna v0.3 (Bidi + ContextJ + ContextO) | swift-idna v0.3 | Medium | Low (rarely-hit Unicode rules) | 11 |
| RS-family JWT algorithms | swift-jwt-verify v0.4 | High | Low — **STILL BLOCKED** on swift-crypto `_RSA` SPI. | n/a (blocked) |
| Crypto-adjacent | various | Medium-low | Low — 22x rejected. | 22 (rejected) |
| Package rename `swift-jwt-verify` → `swift-jwt` | rename project | Low | Low-medium | n/a |

### Recommendation: **Anchor on codec-tier v0.4 drain() API sweep (4-tranche phase)**

Reasons:

1. **Closes Phase 25's documented multi-coding deferral.** swift-content-encoding v0.5 throws `.multipleCodingsNotStreamable` at init explicitly because streaming encoders only emit bytes at `finish()`. Adding `drain() -> Bytes` to all 4 streaming encoders unblocks multi-coding chains.
2. **Completes the streaming HTTP body story (encode side).** Phase 22-25 wired single-coding streaming through the HTTP layer; Phase 27 + Phase 28 close the multi-coding loop.
3. **Small per-tranche scope.** Each codec needs a `drain()` method that:
   - Returns currently-accumulated bytes from the internal `BitWriter`.
   - Resets the internal buffer.
   - Does NOT emit any block terminator (that's `finish()`'s job).
   This is a ~10-20 LOC addition per package + 5-10 tests per package. ~30-60 min per tranche.
4. **Reuses canonical streaming-encoder shape.** Pattern is fully canonical (4/4 instances + content-encoding orchestration); drain() is an additive method that doesn't reshape the type.
5. **Multi-tranche-phase precedent** at Phase 9 (4 tranches: deflate/gzip/zlib/content-encoding v0.2) and Phase 24 (2 tranches: gzip/zlib v0.3). Phase 27 fits the pattern.
6. **Audience continuity.** Same HTTP-codec adopters from Phases 22-25. Strong continuation.
7. **Procedural-correction approaching threshold.** Multi-coding streaming has been deferred 2 times now (Gate 25 codification + Gate 26 ongoing). Phase 27 closes it before the deferral count grows.
8. **Non-breaking additive minor bump per tranche.** All v0.3 APIs unchanged. v0.4 adds:
   - `drain() -> Bytes` method on `Streaming.Encoder` of each codec.
   - Internal: `BitWriter.drain() -> Bytes` (reset buffer; return accumulated bytes).
   - No new error cases needed.
9. **Sets up Phase 28** for swift-content-encoding v0.6 multi-coding streaming wiring (single tranche; would close the v0.6 deferral mentioned in v0.5 CHANGELOG).

### Phase 27 tranche sketch (subject to formal RFC + brainstorm reality-check per tranche)

| Tranche | Package | Scope | Estimated calendar (reality-check-locked) |
|---|---|---|---|
| 27A | swift-brotli v0.4 | `Brotli.Streaming.Encoder.drain() -> Bytes` + ~5 tests | ~30-45 min |
| 27B | swift-deflate v0.4 | `Deflate.Streaming.Encoder.drain() -> Bytes` + ~5 tests | ~30-45 min |
| 27C | swift-gzip v0.4 | `Gzip.Streaming.Encoder.drain() -> Bytes` + ~5 tests | ~30-45 min |
| 27D | swift-zlib v0.4 | `Zlib.Streaming.Encoder.drain() -> Bytes` + ~5 tests | ~30-45 min |

Estimated **total Phase 27 ~2-3 hours wall-clock** for the 4-tranche sweep. Per-tranche brainstorm reality-check should verify:

- `BitWriter`'s internal state can be cleanly split (drain emits accumulated bytes; partial-byte buffer must NOT be byte-aligned because more bits may follow).
- `gzip` + `zlib` streaming encoders wrap `Deflate.Streaming.Encoder` — `drain()` on the outer calls drain on the inner. CRC32/ADLER32 state continues across drain calls (the inner encoder's drain doesn't reset the checksum).
- `drain()` is non-throwing (just returns accumulated bytes) or throws on finished-state (same semantics as update — silent no-op when finished).

**Drain semantics — locked at RFC time:**
- `drain()` is a `mutating` method on `Streaming.Encoder`.
- Returns the accumulated bytes from the internal buffer. The encoder remains in `.open` state.
- **Does NOT byte-align.** Partial-byte buffer state survives drain.
- **Does NOT emit any terminator block.** That's `finish()`'s job.
- Silent no-op on finished encoder (returns empty `Bytes`).
- Multiple drains are valid — bytes flow continuously until `finish()`.

### Why other waves were rejected for Phase 27

- **Streaming decoders sweep** — 3-5hr per tranche × 4 = 12-20hr; biggest investment of any candidate; defer until adopter demand is clearer.
- **Single-codec streaming decode** — would start the decode sweep without completing it; phase commits to a 4-package coordinated sweep is cleaner.
- **swift-oauth2-client v0.3** — v0.2 just shipped same-day (zero settle time). Phase 28+ candidate when v0.2 has 2-3+ days exposure.
- **swift-brotli v0.4 + swift-deflate v0.4 window carry** — could pair with drain sweep in the same v0.4 push; consider integrating into Phase 27.
- **swift-distributed-tracing-bridge v0.3 (B3 + Jaeger)** — no demand.
- **swift-idna v0.3** — 11 deferrals but rarely-hit rules; **not procedural drift** (deferral is correct, not perpetuating-out-of-laziness; adopters don't need Bidi/ContextJ for HTTP/web). Phase 30+ if demand surfaces.
- **RS-family JWT** — still BLOCKED on `_RSA` SPI.
- **Crypto-adjacent** — 22nd rejection.
- **Package rename** — premature.

### Optional Phase 27 expansion: bundle window carry

Phase 27 could OPTIONALLY include window-carry-across-chunks as a fifth tranche (or split into Phase 27 + Phase 28 v0.4):

- 27E (optional): swift-brotli v0.4 + swift-deflate v0.4 window carry across chunks (ratio improvement).

Recommendation: **defer window carry to Phase 28+** (after drain sweep ships) to keep Phase 27 scope tight and per-tranche calendar predictable.

**Action:** author the anchor decision as **RFC-0032** (Phase 27 anchor: codec-tier v0.4 drain() API sweep, 4-tranche phase) and accept it before Phase 27 plans start.

---

## Open work to clear Gate 26

1. **Write RFC-0032** (Phase 27 anchor: codec-tier v0.4 drain() API sweep, 4-tranche). 1-hour task.
2. **Begin Phase 27A brainstorm + reality-check** after RFC-0032 accepts. Brainstorm 27A should:
   - Survey `BitWriter` internals in swift-brotli — confirm partial-byte buffer can be retained across `drain()`.
   - Settle on `drain()` return type (Bytes).
   - Confirm `finish()` semantics unchanged after introducing `drain()`.
   - Settle on tests: drain produces decodable bytes when concatenated with finish; multiple drains preserve total byte equality to single-drain-at-finish.

---

## What this retrospective changes for Phase 27

- The plan-then-execute-inline workflow stays.
- **Wrapper-package vs net-new-features calibration confirmed:** Phase 26 (net-new-features, 1.5hr) fits the 1-2 hour bracket; wrapper packages still fit 30-60 min bracket (Phases 23-25). Phase 27 drain() additions are **wrapper-pattern** (small additive method per package) — expect 30-45 min per tranche.
- **Reality-check before RFC estimate at 5/5** — Phase 27 RFC defers per-tranche estimate to brainstorm.
- **swift-crypto dep canonicalization** — confirmed pattern for crypto-needing packages.
- **CSPRNG-via-stdlib pattern** codified for random-only needs.
- **`@inlinable` + private statics constraint** memory note codified.
- **Auth-tier feature-complete.** Phase 26 closed the auth-tier story. Future auth work (token caching, OIDC helpers, mTLS, private_key_jwt) shifts to v0.3+ candidates as adopter demand surfaces.

---

## Decision

Gate 26 is **PASSED on items 1, 2, 3, 4** and **NOT YET PASSED on item 5 (open)**. Phase 27 plans should be drafted but not executed until item 5 closes via RFC-0032.

Phase 26 completed OAuth 2.0 client coverage. Auth-tier story feature-complete. Codec-tier streaming sweep is the natural next focus area — single-coding done in Phase 25, multi-coding requires the v0.4 drain() API sweep recommended for Phase 27.

The next step is **RFC-0032 anchoring Phase 27 on codec-tier v0.4 drain() API sweep (4-tranche phase: brotli + deflate + gzip + zlib)** — unblocks multi-coding streaming in content-encoding v0.6 (Phase 28+). Streaming decoders sweep, oauth2-client v0.3 (after settle time), brotli + deflate v0.4 window carry, B3 + Jaeger, idna v0.3, RS-family JWT (when swift-crypto stabilizes `_RSA`), and package rename remain on the queue for Phase 28+.

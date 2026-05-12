# RFC-0016 — Phase 11 anchor: Auth primitives wave

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-12 |
| Resolution | Accepted 2026-05-12 — Phase 11 plans may begin authoring. First package: swift-basic-auth. |

## Summary

Anchor Phase 11 on the **auth primitives wave**: ship parser/serializer/verifier packages for the three dominant HTTP authentication schemes (RFC 7617 Basic, RFC 6750 Bearer, RFC 7519 JWT) plus a JWT-claims validator. Four small-to-medium packages, structurally similar to Phase 4's networking-primitives wave (URI / MIME / cookie). Closes the HTTP-server primitive stack assembled across Phases 4 / 7 / 8 / 10.

The wave decomposes into tranches by independence: each package is shippable on its own.

## Problem

[Gate 10 retrospective](../docs/gates/2026-05-12-gate-10-retrospective.md) item 5 requires "Phase 11 anchor decision recorded as an RFC" before Phase 11 plans begin. The retrospective surveyed five candidate waves (auth primitives, internationalization, OTLP cross-signal, swift-brotli v0.2 encoder, crypto-adjacent); this RFC formalizes the choice.

After 39 packages across ten phases, the bare-swift ecosystem provides every primitive an HTTP server needs for request decoding, header parsing, response encoding, and compression — but **no authentication verification primitives**. Server frameworks (Vapor, Hummingbird) bundle their own JWT and Basic Auth implementations, often deeply tied to NIO and Foundation. The Swift-server ecosystem has no Foundation-free, Sendable-clean, framework-agnostic option for auth primitives.

Gate 9's retrospective named auth primitives as the natural Phase 11 anchor once Phase 10 (brotli) closed. Gate 10 confirmed.

## Proposal

### Anchor: auth primitives wave (4 packages)

Phase 11 primary work targets the three dominant HTTP auth schemes plus claim validation:

- **swift-basic-auth (RFC 7617)** — parse and serialize `Authorization: Basic <base64(user:pass)>` headers. ~80 LOC. Smallest deliverable; ships first.
- **swift-bearer (RFC 6750)** — parse and serialize `Authorization: Bearer <token>` headers (plus optional `WWW-Authenticate: Bearer error="..."` for response framing). ~80 LOC.
- **swift-jwt-verify** — verify JWS-compact JWTs (`xxx.yyy.zzz` three-segment Base64URL form) against a pinned public key (RS256 + ES256) or HMAC secret (HS256). Wraps swift-crypto for signature verification; no signing. ~400 LOC.
- **swift-jwt-claims (RFC 7519)** — validate registered claims (`exp`, `nbf`, `iat`, `iss`, `aud`, `sub`, `jti`) per RFC 7519 § 4.1. Uses swift-time for clock + RFC 3339 / Unix timestamp parsing. ~150 LOC.

The whole wave is **~700 LOC source** across four packages — comparable in shape to Phase 4 networking-primitives.

### Tranches

| Tranche | Package | Estimated calendar |
|---|---|---|
| 11A | swift-basic-auth — RFC 7617 Basic | ~0.5 day |
| 11B | swift-bearer — RFC 6750 Bearer | ~0.5 day |
| 11C | swift-jwt-verify — JWS-compact verify (HS256 / RS256 / ES256) | ~1.5 days |
| 11D *(stretch)* | swift-jwt-claims — RFC 7519 registered-claim validation | ~0.5 day |

Total Phase 11 budget: ~1 week wall-clock at observed pace.

### Why this anchor

1. **Closes the HTTP-server primitive stack.** Combined with Phase 4 (URI / MIME / cookie), Phase 7 (compression decoders), Phase 8 (multipart / range / conditional / link-header), Phase 9 (compression encoders), and Phase 10 (brotli), Phase 11's auth primitives complete the request-handling story. After Phase 11, an HTTP server can:
    - decompress the body (Phase 7+10),
    - parse headers (Phase 4+8),
    - **verify identity (Phase 11)**,
    - apply business logic,
    - compress the response (Phase 9),
    - send it back.
2. **Audience continuity is total.** Same HTTP server/client implementers Phases 4 / 7 / 8 / 9 / 10 serve.
3. **Decomposes naturally.** Each package is independently useful. Basic and Bearer auth users don't need JWT; JWT users probably want both Basic and Bearer too. Shipping order: 11A → 11B → 11C → 11D unblocks adopters incrementally.
4. **Risk profile is well-bounded.** Basic and Bearer are syntactic — a parser + a serializer + a typed-throws enum, no algorithmic complexity. JWT verify wraps swift-crypto's `JSONWebSignature` / `P256.Signing` / `HMAC` primitives — we don't reimplement crypto, we wire it into a Foundation-free Swift API. JWT claims is data validation against `Time.Calendar` (swift-time) values.
5. **swift-crypto dep is intentional and isolated.** swift-jwt-verify is the second bare-swift package to wrap swift-crypto (after swift-distributed-tracing-bridge in Phase 3). swift-crypto is explicitly a "wrap-not-reimplement" target per the bare-swift README. Only swift-jwt-verify needs it; basic-auth and bearer don't.
6. **Sets up Phase 12+ options.** Once auth ships:
    - **swift-brotli v0.2 encoder** is the natural Phase 12 anchor.
    - **Internationalization** (IDNA / PSL) becomes approachable.
    - **OTLP cross-signal** stays available.
    - **OAuth 2.0 client primitives** (token endpoint, refresh flow) become composable on top of swift-bearer.

### Tranche 11A scope sketch

`swift-basic-auth` v0.1 covers RFC 7617:

- `BasicAuth.parse(_ headerValue: String) throws(BasicAuthError) -> (username: String, password: String)` — parses `Basic <base64(user:pass)>`. Rejects malformed Base64, missing `:` separator, non-UTF-8 payloads.
- `BasicAuth.serialize(username:password:) -> String` — produces `Basic <base64(user:pass)>`. Rejects usernames containing `:` (per RFC 7617 § 2).
- `BasicAuthError` typed-throws enum.
- Depends on swift-base64 0.1.0 (already in the ecosystem).

### Tranche 11B scope sketch

`swift-bearer` v0.1 covers RFC 6750:

- `Bearer.parseAuthorization(_ headerValue: String) throws(BearerError) -> String` — parses `Authorization: Bearer <token>`. Token is opaque; we don't impose any format constraints (JWTs are one possibility; OAuth opaque tokens are another).
- `Bearer.serializeAuthorization(_ token: String) throws(BearerError) -> String` — produces `Authorization: Bearer <token>`. Rejects tokens containing characters outside the RFC 6750 § 2.1 `b64token` grammar.
- `Bearer.parseWWWAuthenticate(_ headerValue: String) throws(BearerError) -> Challenge` — parses `WWW-Authenticate: Bearer realm="...", error="invalid_token", error_description="..."` response headers (§ 3).
- `Bearer.Challenge` value type carrying `realm`, `error`, `errorDescription`, `errorURI`, `scope` (per § 3).
- `BearerError` typed-throws enum.

### Tranche 11C scope sketch

`swift-jwt-verify` v0.1 covers JWS-compact verification:

- `JWTVerifier(key:algorithm:)` — constructs a verifier from a public key (or HMAC secret) and an explicit algorithm (RS256 / ES256 / HS256).
- `JWTVerifier.verify(_ token: String) throws(JWTVerifyError) -> JWTClaims` — splits the three-segment compact token, decodes the header and payload (Base64URL → bytes; for v0.1 we expose `JWTClaims` as the raw payload bytes since JSON parsing is the caller's responsibility), verifies the signature against the key, and returns the claims.
- `JWTVerifyError` typed-throws enum (malformed token, unsupported algorithm, signature mismatch, etc.).
- `JWTAlgorithm` enum: `.rs256` (RSA-PKCS1-v1.5 SHA-256), `.es256` (ECDSA P-256 SHA-256), `.hs256` (HMAC-SHA256).
- Depends on swift-crypto (wrapped, not reimplemented) and swift-base64.

**Out of scope for v0.1:**
- Signing. Verify-only — we're not a JWT producer.
- Algorithm negotiation. Caller pins the expected algorithm.
- JWK / JWKS endpoints. Caller provides the public key directly.
- ES384 / ES512 / RS384 / RS512 / PS256+ — add in v0.2 if needed.

### Tranche 11D scope sketch (stretch)

`swift-jwt-claims` v0.1 covers RFC 7519 § 4.1:

- `JWTClaimsValidator(now:audience:issuer:)` — constructs a validator with an expected audience, issuer, and a clock source (defaults to `Time.now`).
- `JWTClaimsValidator.validate(_ claims: [String: Any]) throws(JWTClaimsError)` — checks `exp` (not expired), `nbf` (not-yet-active gate), `iat` (issued-at sanity, ≤ now + clockSkew), `iss` (matches expected), `aud` (matches expected), `sub` / `jti` (present if required).
- `JWTClaimsError` typed-throws enum.
- Depends on swift-time (for `Time.Calendar` clock + RFC 3339 / Unix timestamp parsing) and swift-json (for claim payload parsing).

Note: this is the **first bare-swift package whose `claims` parameter is structured input** (it's `[String: Any]` rather than `Bytes`). The design considers using swift-json's `JSONValue` type as the input — to be decided in the implementation plan.

### Out of scope for Phase 11

- **JWT signing.** v0.1 is verify-only across the wave; signing waits for Phase 12+ if there's adopter signal.
- **OAuth 2.0 flows.** Token endpoint, refresh, PKCE, etc. — separate composition over swift-bearer.
- **Digest authentication (RFC 7616).** Deprecated in practice; skip.
- **Mutual TLS / client certs.** Lives in swift-nio territory, not header-layer.
- **swift-brotli v0.2 encoder.** Phase 12 anchor candidate.
- **Internationalization.** Phase 12+ candidate.

### Tooling expectations

Phase 11 reuses all Phase 7+8+9+10 tooling without extension:

- `bare-swift new` for the four new package scaffolds.
- `bare-swift gen-site --umbrella .` after each v0.1 release.
- Pre-emptive Pages tag-policy stays canonical (now 8-for-8 green-on-first-tag).
- Strict-concurrency in CI stays mandatory.
- `docc-target` opt-in from scaffold time.
- Inline test vectors stay the rule; for swift-jwt-verify, cross-library validation against `pyjwt` or `jose` runs at ship time via separate consumer scripts (NOT in the test target — Foundation-free invariant per Gate 8/9 standing memory).

### Versioning

Each new package gets a v0.1.0 tag:

| Package | New | Future |
|---|---|---|
| swift-basic-auth | v0.1 (parser + serializer) | v0.2 if needed (likely stable at v0.1) |
| swift-bearer | v0.1 (parser + serializer + WWW-Authenticate) | v0.2 if needed |
| swift-jwt-verify | v0.1 (RS256/ES256/HS256 verify) | v0.2 = signing + additional algs |
| swift-jwt-claims | v0.1 (RFC 7519 registered claims) | v0.2 if custom-claim extensions surface |

Phase 11 introduces **4 new packages**, taking the ecosystem from 39 to **43 packages**.

### Anti-goals

- **No new tier on the umbrella.** Auth packages join the existing tiers — basic-auth + bearer in "http"; jwt-verify + jwt-claims either there or in a new "auth" sub-section (TBD by `gen-site` tier-grouping). Decision deferred to the executor.
- **No swift-crypto in baseline packages.** Only swift-jwt-verify needs it; basic-auth and bearer must NOT depend on swift-crypto.
- **No re-implementation of cryptographic primitives.** RS256 / ES256 / HS256 verification all wrap swift-crypto's audited code.
- **No retroactive bug-fix sweep across prior phases.**

## Alternatives considered

### Anchor Phase 11 on swift-brotli v0.2 (encoder)

**Pros:** closes Phase 10's decoder-first staging; encoder is anchor-worthy LOC (~1500).

**Cons:** swift-brotli decoder just shipped; encoder tuning is its own focused effort better suited to a dedicated single-package phase like Phase 12. Also, swift-brotli v0.2 doesn't expand the ecosystem's reach (existing audience already has the decoder); auth primitives close a real gap.

**Verdict:** rejected as Phase 11. Strong Phase 12 candidate.

### Anchor Phase 11 on internationalization (swift-idna, swift-publicsuffix)

**Pros:** closes swift-uri's IDNA deferral; underrepresented.

**Cons:** IDNA is famously fiddly (UTS #46 + UTS #15 + Punycode); PSL is data-heavy (~150 KiB). Two packages of comparable density to swift-brotli v0.1. Probably an entire phase on its own. Better suited to dedicated focus once auth is done.

**Verdict:** rejected as Phase 11. Phase 12+ candidate.

### Anchor Phase 11 on OTLP cross-signal v0.2

**Pros:** closes Phase 2/3/4 single-signal deferral.

**Cons:** narrower audience than auth primitives. Three packages of medium complexity, each requiring protobuf encoder updates.

**Verdict:** rejected as Phase 11. Phase 13+ candidate.

### Skip auth, do crypto-adjacent (swift-blake3 / swift-siphash / swift-hkdf)

**Pros:** would complement swift-jwt-verify if it ships.

**Cons:** swift-crypto remains entrenched. Eighth rejection candidate across gates. Differentiation hasn't strengthened.

**Verdict:** rejected.

## Resolution

**Accepted 2026-05-12.** Phase 11 plans may begin authoring. First package: swift-basic-auth.

Phase 11 closes the HTTP-server primitive stack with authentication-verification primitives. Phase 12 anchor decision (likely swift-brotli v0.2 encoder or internationalization) will land at Gate 11.

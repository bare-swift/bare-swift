# RFC-0024 — Phase 19 anchor: swift-jwt-verify v0.2 (signing — HS256 + ES256)

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-16 |
| Resolution | Accepted 2026-05-16 — Phase 19 plans may begin authoring. Single existing package: swift-jwt-verify v0.2. |

## Summary

Anchor Phase 19 on **swift-jwt-verify v0.2** — additive minor version bump adding `JWTSigner.signHS256(_:key:)` and `JWTSigner.signES256(_:key:)` alongside the existing v0.1 verify entry points. Algorithm scope matches v0.1's verification surface exactly (HS256 + ES256 via swift-crypto). Closes the longest-deferred functional gap in the auth tier — the verify-only design boundary that has been pushed forward for 7 consecutive gate retrospectives.

Single-tranche, additive-only minor bump. Non-breaking — v0.1 verify APIs unchanged. Same shape as Phase 18's swift-distributed-tracing-bridge v0.2 bump.

## Problem

[Gate 18 retrospective](../docs/gates/2026-05-14-gate-18-retrospective.md) item 5 requires "Phase 19 anchor decision recorded as an RFC" before Phase 19 plans begin. The retrospective surveyed seven candidate waves; this RFC formalizes the choice.

**Concrete gap:** swift-jwt-verify v0.1 (Phase 11C, shipped 2026-05-12) is a verify-only library. Adopters who consume JWTs (resource servers validating bearer tokens, OAuth clients validating ID tokens) get clean ergonomic verification via `JWTVerifier.verifyHS256(_:key:)` / `JWTVerifier.verifyES256(_:publicKey:)`. **Adopters who issue JWTs (authorization servers, identity providers, internal token mints) have no signing primitive** — they have to drop down to swift-crypto and hand-assemble the JWS Compact format (`base64url(header) "." base64url(payload) "." base64url(signature)`) themselves.

This gap has been deferred at every gate since Phase 11 closed:

| Gate | Retro line |
|---|---|
| 11 | "fresh v0.1; let it settle" |
| 12 | "rejected as Phase 13; v0.1 has had ~1 day" |
| 13 | "Phase 17+ when v0.1 has had a week" |
| 14 | "Phase 17+ candidate" |
| 15 | "v0.1 has had only ~1 day" |
| 16 | "fresh v0.1 (~2 days)" |
| 17 | "v0.1 has had ~2 days" |
| 18 | **8th rejection threshold** — "wait a week" criterion has been driving deferral for 4 days |

The "wait a week" criterion was reasonable when v0.1 was fresh. v0.1 has now had 4 days of settle time across the trinity-focus phases (14B/14C through 18A) with no reported issues. Continuing to defer mechanically against a self-imposed clock when no actual blocker exists is procedural drift. **Phase 19 closes the gap.**

After Phases 11 / 12 / 13 / 14 / 15 / 16 / 17 / 18, the auth tier is at v0.1 feature-completion **except for the issuer-side signing primitive**. Phase 19 closes it.

## Proposal

### Anchor: swift-jwt-verify v0.2 (single existing package)

**swift-jwt-verify v0.2** — add JWS Compact signing primitives matching v0.1's verification algorithm surface.

| API addition | Description |
|---|---|
| `JWTSigner.signHS256(_:key:) -> String` | Sign claims (passed as `[UInt8]` JSON-encoded payload) with HMAC-SHA-256 using a symmetric key. Returns JWS Compact serialization. |
| `JWTSigner.signES256(_:privateKey:) -> String` | Sign claims with ECDSA P-256 SHA-256 using a `P256.Signing.PrivateKey` from swift-crypto. Returns JWS Compact serialization. |
| (header construction internals) | `{"alg":"HS256","typ":"JWT"}` and `{"alg":"ES256","typ":"JWT"}` headers built canonically. |

**Dependency:** swift-crypto (already in v0.1; no version bump needed).

**Non-breaking:** all v0.1 verify entry points (`JWTVerifier.verifyHS256` / `JWTVerifier.verifyES256`) unchanged. v0.2 only adds new top-level functions on a new `JWTSigner` namespace alongside the existing `JWTVerifier`.

### Tranches

Single-anchor phase shape, two reasonable decompositions:

**Shape A (recommended): single tranche, HS256 + ES256 signing**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 19A | swift-jwt-verify v0.2 | `JWTSigner.signHS256` + `JWTSigner.signES256` + JWS Compact serializer + 12-15 round-trip tests | ~0.5-1 day |

**Shape B (alternate): two tranches if EC scope expands**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 19A | swift-jwt-verify v0.2 | HS256 signing only | ~0.3 day |
| 19B | swift-jwt-verify v0.3 | ES256 signing (if EC key marshalling exceeds budget) | ~0.5 day |

Decision deferred to brainstorm phase. Per the now-canonical brainstorm-reality-check pattern, expect Shape B if swift-crypto's `P256.Signing.PrivateKey` external surface (PKCS#8 / SEC1 / raw key bytes) is wider than expected. swift-crypto's `P256` types are stable and well-documented; Shape A is the likely outcome.

### Tranche 19A scope sketch

`swift-jwt-verify` v0.2 covers:

- **`JWTSigner` namespace** — sibling of v0.1's `JWTVerifier` namespace. Public enum or `Sendable` struct (final choice in brainstorm; matches v0.1's `JWTVerifier` shape).
- **`JWTSigner.signHS256(payload:key:) -> String`** — accepts `[UInt8]` payload (JSON-encoded claims; the caller produces this) and `[UInt8]` symmetric key. Returns JWS Compact string. Internally:
  1. Construct canonical header bytes `{"alg":"HS256","typ":"JWT"}`.
  2. Base64URL encode header → `h`.
  3. Base64URL encode payload → `p`.
  4. Compute `HMAC<SHA256>` over `h + "." + p` using the key.
  5. Base64URL encode HMAC tag → `s`.
  6. Return `h + "." + p + "." + s`.
- **`JWTSigner.signES256(payload:privateKey:) -> String`** — same flow but ECDSA P-256:
  1. Header bytes `{"alg":"ES256","typ":"JWT"}`.
  2. Compute SHA-256 over `h + "." + p`.
  3. Sign hash with `P256.Signing.PrivateKey.signature(for:)`.
  4. Convert the signature to **JWS R||S concatenated raw 64-byte format** (per RFC 7518 § 3.4) — `swift-crypto`'s `P256.Signing.ECDSASignature.rawRepresentation` is exactly this 64-byte form.
  5. Base64URL encode → `s`.
  6. Return `h + "." + p + "." + s`.
- **JWS Compact serializer helper** — private, reused by both sign functions. Takes the three byte arrays (header, payload, signature) and joins with `.`.
- **No new errors.** Signing operations on validated inputs (well-formed payload bytes + correct key length) shouldn't fail. swift-crypto's signature operation is total on valid keys.

**Reuses existing types:**
- `Base64URL.encode(_:)` / `Base64URL.decode(_:)` from swift-jwt-verify v0.1.
- `HMAC<SHA256>` / `P256.Signing.PrivateKey` from swift-crypto (v0.1 dependency).

**Round-trip tests (the strict invariant):**
- HS256 sign → v0.1's HS256 verify must return success with identical claims.
- ES256 sign → v0.1's ES256 verify must return success with identical claims.
- Cross-verify against RFC 7515 Appendix A.1 (HS256) and A.3 (ES256) reference vectors.

**Out of scope for v0.2:**
- **RS256 (RSA-SHA-256).** Different key marshalling (PEM / PKCS#1 / PKCS#8); v0.3 candidate if adopter demand surfaces.
- **ES384 / ES512.** Different curve families; v0.3 candidate.
- **EdDSA (Ed25519).** Modern algorithm; v0.3 candidate.
- **Header customization.** v0.2 emits canonical `{"alg":"...", "typ":"JWT"}`. Custom `kid` (key ID) / `jku` / `x5u` headers deferred to v0.3.
- **Claim builder helpers.** v0.2 accepts `[UInt8]` payload — caller produces JSON. A `JWTClaims` struct + JSON encoder is a v0.3+ concern.
- **JWE (encryption).** Out of scope entirely.
- **JWK / JWKS handling.** Out of scope entirely.

### Why this anchor

1. **Closes the longest-standing functional gap.** 7 prior gate rejections; 8th would be procedural drift.
2. **Additive, non-breaking minor bump.** Same low-risk shape as Phase 18.
3. **Algorithm scope matches v0.1 verification.** No new swift-crypto integrations beyond what v0.1 already uses. ~250-400 LOC estimate.
4. **High value-to-LOC ratio.** Unlocks the auth tier for issuer-side adopters.
5. **Audience continuity.** Auth/security adopters from Phase 11 (swift-basic-auth + swift-bearer + swift-jwt-verify).
6. **Net new community contact.** Phases 14-18 focused on observability; Phase 19 returns to the auth tier from Phases 11A/B/C.
7. **Risk profile is well-bounded.**
   - swift-crypto's `HMAC<SHA256>` provides `init(authenticating:using:)` — symmetric verification primitive that v0.1 already uses; signing is the same primitive used in reverse direction.
   - swift-crypto's `P256.Signing.PrivateKey` provides `.signature(for:)` returning `P256.Signing.ECDSASignature` — `rawRepresentation` is exactly the 64-byte JWS R||S format.
   - JWS Compact format is RFC 7515-specified. Reference test vectors in Appendix A.
   - **Round-trip tests against v0.1's verifier are the unambiguous correctness gate** — anything that signs and round-trips through v0.1's verifier is by definition compliant.
8. **Sets up Phase 20+ options.**
   - **RS256** is the natural v0.3 expansion candidate.
   - **OAuth 2.0 client primitives** become more relevant once swift-jwt-verify supports both sign + verify (OAuth issuer + relying-party flows both need it).
   - **swift-distributed-tracing-bridge v0.3 (B3 + Jaeger)** stays available.
   - **swift-idna v0.3 / swift-brotli v0.3** stay available.

### Out of scope for Phase 19

- **RS256 (RSA).** Phase 20+ candidate.
- **ES384, ES512, EdDSA (Ed25519).** Phase 20+ candidate.
- **Header customization (kid, jku, x5u).** v0.3.
- **Claim builder helpers / JSON encoder.** v0.3+.
- **JWE / JWK / JWKS.** Out of scope entirely.
- **swift-distributed-tracing-bridge v0.3 / swift-idna v0.3 / swift-brotli v0.3 / OAuth 2.0.** Phase 20+ candidates.

### Tooling expectations

Phase 19 reuses all Phase 7–18 tooling without extension:

- Existing swift-jwt-verify repo (no scaffolding).
- Strict-concurrency in CI stays mandatory.
- `docc-target` already set.
- Sanitizers ON (no large static data).
- Cross-library validation at ship time: round-trip sign → verify within the package; cross-check against RFC 7515 Appendix A test vectors.

### Versioning

| Package | New | Future |
|---|---|---|
| swift-jwt-verify | v0.2 (HS256 + ES256 signing) | v0.3 = RS256 + ES384/ES512 + EdDSA + header customization + claim builder if adopter demand |

Phase 19 introduces **0 new packages** (existing-package minor bump). Ecosystem stays at **48 packages**.

### Renaming consideration

The package is named `swift-jwt-verify`. After v0.2 ships signing, the name is technically misleading. **Decision: do not rename in v0.2.** Renaming requires:
- New repo at `bare-swift/swift-jwt` (or similar).
- Migration period with both repos shipping briefly.
- Adopter SemVer churn (`swift-jwt-verify` deprecated, point at new repo).

Phase 19 is an additive minor bump on the existing package. A rename to `swift-jwt` is a Phase 20+ candidate if/when the package's verify+sign surface justifies the migration cost. For v0.2, the README + DocC clarify that v0.2+ supports signing despite the package name.

### Anti-goals

- **No RSA / EdDSA / ES384 / ES512.** Phase 20+.
- **No header customization.** Canonical `{"alg":"...", "typ":"JWT"}` only.
- **No claim builder.** Adopter produces `[UInt8]` JSON payload.
- **No retroactive changes to v0.1 verify APIs.** Purely additive.
- **No package rename.** v0.2 stays at `swift-jwt-verify` despite the verify-only name.

## Alternatives considered

### Anchor Phase 19 on swift-distributed-tracing-bridge v0.3 (B3 + Jaeger propagation)

**Pros:** natural follow-on to v0.2 W3C; would round out trace-propagation format coverage.

**Cons:** no concrete adopter demand for B3/Jaeger when W3C just landed. Third v0.x bump on the bridge in two phases would be aggressive churn. Phase 20+ when demand surfaces.

**Verdict:** rejected as Phase 19.

### Anchor Phase 19 on swift-idna v0.3 (Bidi + ContextJ + ContextO)

**Pros:** closes Phase 15's documented deferral.

**Cons:** rarely-hit rules; no adopter demand. Phase 20+ candidate.

**Verdict:** rejected as Phase 19.

### Anchor Phase 19 on swift-brotli v0.3 (streaming encoder)

**Pros:** closes Phase 12 v0.2 one-shot-only deferral.

**Cons:** narrower audience than closing the auth tier's verify-only gap.

**Verdict:** rejected as Phase 19.

### Anchor Phase 19 on OAuth 2.0 client primitives

**Pros:** composes on top of swift-bearer + swift-jwt-verify.

**Cons:** OAuth 2.0 has many flows; picking the right v0.1 subset needs adopter input. **Also: OAuth issuer flows benefit from JWT signing being available, so JWT signing should land first.**

**Verdict:** rejected as Phase 19. Phase 20+ candidate after signing lands.

### Skip ahead, do crypto-adjacent (swift-blake3 / swift-siphash / swift-hkdf)

**Pros:** would complement JWT signing.

**Cons:** swift-crypto remains entrenched. **15th consecutive rejection.**

**Verdict:** rejected.

### Anchor Phase 19 on swift-publicsuffix v0.2 (ICANN/PRIVATE split)

**Pros:** closes a quiet Phase 13 deferral.

**Cons:** narrow scope. Better as a v0.1.1 patch.

**Verdict:** rejected as Phase 19 anchor.

### Bundle JWT signing with package rename to swift-jwt

**Pros:** name reflects new capability.

**Cons:** rename has migration cost; better to ship the feature first and rename later if/when adopter cognitive overhead from the misleading-name justifies the SemVer churn.

**Verdict:** rejected for v0.2. Phase 20+ candidate if adopter feedback surfaces.

## Resolution

**Accepted 2026-05-16.** Phase 19 plans may begin authoring. Single existing package: swift-jwt-verify v0.2.

Phase 19 closes the longest-standing functional gap in the auth tier — the verify-only design boundary that has been deferred for 7 consecutive gates. After Phase 19, adopters can both verify JWTs from issuers AND issue their own JWTs (authorization servers, identity providers, internal token mints) using the same package and the same algorithm surface (HS256 + ES256).

swift-distributed-tracing-bridge v0.3 (B3 + Jaeger), swift-idna v0.3, swift-brotli v0.3 streaming, RS256, OAuth 2.0, and a potential `swift-jwt` package rename remain on the queue for Phase 20+.

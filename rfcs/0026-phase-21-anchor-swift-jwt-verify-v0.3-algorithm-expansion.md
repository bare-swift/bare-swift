# RFC-0026 — Phase 21 anchor: swift-jwt-verify v0.3 (ES384 + ES512 + EdDSA + `kid` header)

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-16 |
| Resolution | Accepted 2026-05-16 — Phase 21 plans may begin authoring. Single existing package: swift-jwt-verify v0.3. |

## Summary

Anchor Phase 21 on **swift-jwt-verify v0.3** — additive non-breaking minor version bump adding three more JWT algorithms (ES384 + ES512 + EdDSA) and optional `kid` header customization. Algorithm expansion follows v0.2's HS256 + ES256 Signer/Verifier pattern exactly; `kid` is a small additive on top. Closes the algorithm-completion deferral documented in v0.2's CHANGELOG. Non-breaking — all v0.2 APIs unchanged.

Single-tranche; ~300-500 LOC; estimated ~1-2 hours wall-clock per the systematic-calendar-estimate calibration noted in Gate 20.

## Problem

[Gate 20 retrospective](../docs/gates/2026-05-16-gate-20-retrospective.md) item 5 requires "Phase 21 anchor decision recorded as an RFC" before Phase 21 plans begin. The retrospective surveyed seven candidate waves; this RFC formalizes the choice.

**Concrete gap:** swift-jwt-verify v0.2 (Phase 19A, shipped 2026-05-16) supports HS256 + ES256 sign + verify. The package's v0.2 CHANGELOG explicitly lists "RS256 / RS384 / RS512 / PS256 / ES384 / ES512 / EdDSA. Phase 20+ candidates." Phase 21 closes three of these:

- **ES384** (ECDSA P-384 + SHA-384) — common in enterprise OIDC; financial-sector compliance (PCI-DSS, FAPI) often mandates P-384 or stronger.
- **ES512** (ECDSA P-521 + SHA-512) — used in high-security contexts.
- **EdDSA** (Ed25519 — RFC 8037) — modern, fast, side-channel-resistant; increasingly favored over ECDSA in new deployments.

RSA-family algorithms (RS256/RS384/RS512/PS256) **remain deferred** to Phase 22+: swift-crypto's RSA API is still under the `_RSA` experimental SPI. Exposing it to adopters before stabilization risks v0.3 → v0.4 breaking churn when swift-crypto promotes the API.

**Why now (Phase 21):**
1. Phase 19 shipped JWT signing for HS256 + ES256. The algorithm-expansion gap was the documented v0.3 deferral.
2. Phase 20 shipped swift-oauth2-client v0.1. OIDC adopters (the primary swift-oauth2-client audience) often need ES384/ES512/EdDSA for compliance or modern-security reasons.
3. Audience continuity is high: same auth/security adopters from Phases 11 + 19 + 20.
4. Scope is bounded and pattern is locked: same Signer/Verifier shape as v0.2; three new algorithms slot in cleanly.

**Why `kid` header in the same phase:**
- Tiny additive (~30 LOC).
- Pairs naturally with algorithm expansion: adopters using multiple algorithms or rotating keys need `kid` to select the right verification key.
- Closes another documented v0.2 deferral ("header customization (kid, jku, x5u, cty) for JWS").
- Avoids a separate v0.4 bump just for one additive field.

## Proposal

### Anchor: swift-jwt-verify v0.3 (single existing package, additive minor bump)

**swift-jwt-verify v0.3** — extend `JWTAlgorithm` enum, `Signer`, `Verifier`, and `JWTSigner` with three more algorithms and one optional header field.

| Addition | Description |
|---|---|
| `JWTAlgorithm.es384` | ECDSA P-384 + SHA-384. RFC 7518 § 3.4. |
| `JWTAlgorithm.es512` | ECDSA P-521 + SHA-512. RFC 7518 § 3.4. |
| `JWTAlgorithm.eddsa` | EdDSA via Ed25519. RFC 8037. |
| `Signer.signES384` / `Verifier.verifyES384` | swift-crypto `P384.Signing.PrivateKey` / `.PublicKey`. |
| `Signer.signES512` / `Verifier.verifyES512` | swift-crypto `P521.Signing.PrivateKey` / `.PublicKey`. |
| `Signer.signEdDSA` / `Verifier.verifyEdDSA` | swift-crypto `Curve25519.Signing.PrivateKey` / `.PublicKey`. |
| `JWTSigner.kid: String?` | Optional `kid` header field; when non-nil, emit `{"alg":"...","typ":"JWT","kid":"..."}` instead of the v0.2 canonical header. |
| `JWTAlgorithm.rfc7518Name` | Extended to return `"ES384"` / `"ES512"` / `"EdDSA"` for the new cases. |
| `JWTAlgorithm.fromName(_:)` | Extended to recognize `"ES384"` / `"ES512"` / `"EdDSA"`. |

### Tranches

Single-anchor phase shape:

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 21A | swift-jwt-verify v0.3 | 3 new algorithms (ES384, ES512, EdDSA) + `kid` header + ~25-30 tests | ~1-2 hours |

Realistic wall-clock estimate per Gate 20's calendar-calibration observation: ~1-2 hours matching Phase 19's ~1 hour and Phase 20's ~1 hour for similar-scope additive bumps.

### Tranche 21A scope sketch

In `Sources/JWTVerify/JWTAlgorithm.swift`, extend the enum:

```swift
public enum JWTAlgorithm: Sendable, Equatable, Hashable {
    case hs256
    case es256
    case es384   // NEW
    case es512   // NEW
    case eddsa   // NEW

    public var rfc7518Name: String {
        switch self {
        case .hs256: return "HS256"
        case .es256: return "ES256"
        case .es384: return "ES384"
        case .es512: return "ES512"
        case .eddsa: return "EdDSA"
        }
    }
}

extension JWTAlgorithm {
    static func fromName(_ name: String) -> JWTAlgorithm? {
        switch name {
        case "HS256": return .hs256
        case "ES256": return .es256
        case "ES384": return .es384
        case "ES512": return .es512
        case "EdDSA": return .eddsa
        default: return nil
        }
    }
}
```

In `Sources/JWTVerify/Signer.swift`, append three new static functions:

```swift
extension Signer {
    static let headerES384: [UInt8] = Array(#"{"alg":"ES384","typ":"JWT"}"#.utf8)
    static let headerES512: [UInt8] = Array(#"{"alg":"ES512","typ":"JWT"}"#.utf8)
    static let headerEdDSA: [UInt8] = Array(#"{"alg":"EdDSA","typ":"JWT"}"#.utf8)

    static func signES384(signingInput: [UInt8], privateKey: [UInt8]) throws(JWTVerifyError) -> [UInt8] {
        // 48-byte raw P-384 scalar → P384.Signing.PrivateKey(rawRepresentation:) →
        // .signature(for: Data(signingInput)).rawRepresentation (96-byte R||S).
        // ...
    }

    static func signES512(signingInput: [UInt8], privateKey: [UInt8]) throws(JWTVerifyError) -> [UInt8] {
        // 66-byte raw P-521 scalar → P521.Signing.PrivateKey(...) → 132-byte R||S.
        // ...
    }

    static func signEdDSA(signingInput: [UInt8], privateKey: [UInt8]) throws(JWTVerifyError) -> [UInt8] {
        // 32-byte raw Ed25519 seed → Curve25519.Signing.PrivateKey(...) → 64-byte signature.
        // ...
    }
}
```

In `Sources/JWTVerify/Verifier.swift`, append three new static functions mirroring v0.2's `verifyES256` shape (using `P384.Signing.PublicKey.x963Representation` / `P521.Signing.PublicKey.x963Representation` / `Curve25519.Signing.PublicKey.rawRepresentation` — note Curve25519 uses raw representation, not x963).

In `Sources/JWTVerify/JWTVerify.swift`, extend `JWTSigner`:

```swift
public struct JWTSigner: Sendable {
    public let algorithm: JWTAlgorithm
    public let key: [UInt8]
    public let kid: String?   // NEW (default nil; v0.2 behavior preserved)

    public init(algorithm: JWTAlgorithm, key: [UInt8], kid: String? = nil) {
        self.algorithm = algorithm
        self.key = key
        self.kid = kid
    }

    public func sign(_ payload: [UInt8]) throws(JWTVerifyError) -> String {
        let headerBytes: [UInt8]
        if let kid {
            // Build {"alg":"...","typ":"JWT","kid":"..."} dynamically.
            // ...
        } else {
            // Use the canonical static header constants.
            // ...
        }
        // ... rest of v0.2's sign() flow with switch extended for ES384/ES512/EdDSA ...
    }
}
```

Update `JWTSigner.sign(_:)` switch to dispatch to the new `Signer.sign*` functions. Extend `JWTVerifier.verify(_:)` similarly.

### Key bytes per algorithm

| Algorithm | Sign key (raw) | Verify key |
|---|---|---|
| `.hs256` | HMAC secret (any length) | Same as sign |
| `.es256` | 32-byte P-256 raw scalar | 65-byte P-256 x963 uncompressed |
| `.es384` | **48-byte P-384 raw scalar** | **97-byte P-384 x963 uncompressed** (`0x04 \|\| 48B x \|\| 48B y`) |
| `.es512` | **66-byte P-521 raw scalar** | **133-byte P-521 x963 uncompressed** (`0x04 \|\| 66B x \|\| 66B y`) |
| `.eddsa` | **32-byte Ed25519 seed** | **32-byte Ed25519 public key** (raw, not x963) |

Adopters derive each pair via swift-crypto:
```swift
let privEs384 = P384.Signing.PrivateKey()
let signerKey = Array(privEs384.rawRepresentation)            // 48 bytes
let verifierKey = Array(privEs384.publicKey.x963Representation) // 97 bytes
```

EdDSA key sizes are notably smaller (32 bytes each) and symmetric in length — a memorable difference vs ECDSA's asymmetric key sizes.

### Tests (~25-30 new tests)

Mirroring v0.2's `SignerTests` + the existing v0.1 verifier tests:

- **AlgorithmTests** (3 new): `rfc7518Name` for each new algorithm; `fromName` round-trip.
- **ES384 sign tests** (5): round-trip; wrong key length (47/49 bytes); zero scalar; signature length is 96 bytes; deterministic.
- **ES512 sign tests** (5): round-trip; wrong key length (65/67 bytes); zero scalar; signature length is 132 bytes; deterministic.
- **EdDSA sign tests** (5): round-trip; wrong key length (31/33 bytes); zero scalar; signature length is 64 bytes; symmetric key sizes.
- **ES384/ES512/EdDSA verify tests** (3): each algorithm via the v0.1-style `JWTVerifier.verify(_:)` path.
- **`kid` header tests** (4): `kid: nil` produces v0.2 canonical header; `kid: "key-1"` produces header with `"kid":"key-1"`; verifier ignores `kid` (doesn't affect verification); cross-algorithm `kid` works.
- **v0.2 tests (33) unchanged** — all 33 v0.2 tests remain green; no breaking changes.

Total: ~25 new tests. Estimated package total: 33 (v0.2) + 25 = ~58 tests across ~8-9 suites.

### Anti-goals for v0.3 (hardcoded)

- **No RS256 / RS384 / RS512 / PS256.** swift-crypto's RSA API still in `_RSA` experimental SPI. Deferred to v0.4 when swift-crypto stabilizes.
- **No `jku` / `x5u` / `cty` / `crit` / `typ` headers.** Only `kid` in v0.3 (most-requested). Other headers v0.4+.
- **No JWE.** Out of scope entirely.
- **No JWK / JWKS endpoint fetching.** Caller supplies raw key bytes.
- **No claim builder helpers.** Caller produces `[UInt8]` JSON payload.
- **No package rename.** v0.3 stays as `swift-jwt-verify`. Rename consideration deferred to Phase 23+.
- **No changes to v0.2 API surface.** Purely additive.

### Out of scope for Phase 21 (other waves)

- **swift-oauth2-client v0.2** — v0.1 just shipped same-day; v0.2 helpers (auth URL builder + PKCE generation) deferred to Phase 22+ when v0.1 has had a few days of settle time.
- **swift-brotli v0.3 streaming encoder** — 8 prior deferrals; strong Phase 22 candidate per the procedural-correction pattern.
- **swift-distributed-tracing-bridge v0.3 (B3 + Jaeger)** — no concrete demand; W3C just landed. Phase 22+.
- **swift-idna v0.3 (Bidi + ContextJ)** — rarely-hit rules; no demand. Phase 22+.
- **Package rename `swift-jwt-verify` → `swift-jwt`** — Phase 23+ after v0.3 algorithm completion makes the rename more compelling.

### Tooling expectations

Phase 21 reuses all Phase 7–20 tooling without extension:

- Existing swift-jwt-verify repo (no new scaffolding).
- Strict-concurrency in CI stays mandatory.
- `docc-target` already set.
- Sanitizers ON (no large static data).
- Cross-library validation at ship time: round-trip sign → verify within the package for each new algorithm.

### Versioning

| Package | New | Future |
|---|---|---|
| swift-jwt-verify | v0.3 (ES384 + ES512 + EdDSA + kid) | v0.4 = RS256/RS384/RS512/PS256 when swift-crypto stabilizes RSA; v0.5 = `jku` / `x5u` / `cty` headers + JWK loading if adopter demand |

Phase 21 introduces **0 new packages** (existing-package minor bump). Ecosystem stays at **49 packages**.

### Calendar estimate

**~1-2 hours wall-clock.** Per Gate 20's systematic-calendar-estimate calibration: focused additive bumps with ~300-500 LOC scope and well-understood primitives reliably ship in ~1-2 hours.

### Anti-goals

- **No swift-crypto RSA SPI exposure.** RS-family deferred.
- **No package rename.**
- **No retroactive changes to v0.1/v0.2 verify/sign APIs.**
- **No JWK/JWKS/JWE.**
- **No new umbrella tier.**
- **No new dependencies.** swift-crypto already in v0.1.

## Alternatives considered

### Anchor Phase 21 on swift-oauth2-client v0.2 (auth URL builder + PKCE + state/nonce)

**Pros:** natural follow-on to Phase 20; closes documented v0.1 deferrals.

**Cons:** v0.1 shipped same-day; "wait for adopter signal" criterion here ISN'T procedural drift (v0.1 hasn't had real-world exposure). Phase 22+ when v0.1 has had several days.

**Verdict:** rejected as Phase 21.

### Anchor Phase 21 on swift-brotli v0.3 (streaming encoder)

**Pros:** 8 prior deferrals matching OAuth's count when it broke; procedural-correction pattern applies.

**Cons:** narrower audience than auth-tier algorithm completion. Phase 21 priority goes to JWT algorithm expansion because audience continuity from Phases 19 + 20 is tighter.

**Verdict:** rejected as Phase 21. Strong Phase 22 candidate.

### Anchor Phase 21 on swift-distributed-tracing-bridge v0.3 (B3 + Jaeger)

**Pros:** rounds out propagation format coverage.

**Cons:** no concrete demand. Phase 22+ candidate.

**Verdict:** rejected as Phase 21.

### Anchor Phase 21 on swift-idna v0.3 (Bidi + ContextJ + ContextO)

**Pros:** closes Phase 15's documented deferral.

**Cons:** rarely-hit rules; no demand. Phase 22+ candidate.

**Verdict:** rejected as Phase 21.

### Skip ahead, do crypto-adjacent (swift-blake3 / swift-siphash / swift-hkdf)

**Pros:** would complement JWT and OAuth.

**Cons:** swift-crypto remains entrenched. **17th consecutive rejection.**

**Verdict:** rejected.

### Anchor Phase 21 on package rename swift-jwt-verify → swift-jwt

**Pros:** v0.3 algorithm completion would make the rename more compelling.

**Cons:** rename has migration cost; better to ship v0.3 features first, then rename in a separate phase. Pre-rename, the v0.3 features are easier to slot into the existing repo. Post-rename, both repos would need to ship in parallel briefly. **Sequence the rename for Phase 23+ after Phase 21 + 22.**

**Verdict:** rejected as Phase 21 anchor.

### Bundle v0.3 algorithm expansion with package rename

**Pros:** one migration for adopters covers both algorithm expansion + name change.

**Cons:** combining substantive features with a rename creates compound risk — if adopters hit a v0.3 bug, the rename migration complicates rollback. Keep them separate.

**Verdict:** rejected; single-anchor-phase shape preserved.

### Bundle algorithm expansion with `jku` / `x5u` / `cty` headers

**Pros:** "complete the JWS header story" in one ship.

**Cons:** `jku` and `x5u` are URL-based and benefit from swift-uri integration; `cty` is rarely used; only `kid` is widely needed. Scope creep. Defer `jku` / `x5u` / `cty` to v0.4+.

**Verdict:** rejected; v0.3 ships only `kid`.

## Resolution

**Accepted 2026-05-16.** Phase 21 plans may begin authoring. Single existing package: swift-jwt-verify v0.3.

Phase 21 closes the documented JWT algorithm-completion deferral for ECDSA + EdDSA families. After Phase 21, swift-jwt-verify supports HS256 + ES256 + ES384 + ES512 + EdDSA sign+verify with optional `kid` header customization — covering the majority of real-world JWT algorithm use cases.

RSA-family algorithms (RS256/RS384/RS512/PS256), swift-oauth2-client v0.2 (auth URL builder + PKCE helpers), swift-brotli v0.3 streaming, swift-distributed-tracing-bridge v0.3 (B3 + Jaeger), swift-idna v0.3, and a possible `swift-jwt` package rename remain on the queue for Phase 22+.

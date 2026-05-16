# swift-jwt-verify v0.2 (HS256 + ES256 signing) Design

**Phase 19 Tranche 19A** (RFC-0024). Existing package; additive non-breaking minor version bump.

**Date:** 2026-05-16

---

## Goal

Add JWS-compact signing primitives matching v0.1's verification algorithm surface exactly (HS256 + ES256 via swift-crypto). Closes the verify-only design boundary that has been deferred for 7 consecutive gate retrospectives. Adopters who issue JWTs (authorization servers, identity providers, internal token mints) get a clean signing primitive alongside v0.1's verifier.

## Scope (Shape A — single tranche, HS256 + ES256)

**In scope:**
- Public `JWTSigner` struct (mirrors v0.1's `JWTVerifier` shape: `algorithm: JWTAlgorithm`, `key: [UInt8]`, non-throwing init, throwing `sign(_:)`).
- Internal `Signer` namespace with `signHS256` + `signES256` static functions (mirrors `Verifier`).
- `Base64URL.encode(_:)` addition (wraps swift-base64's `.urlSafeNoPadding`).
- Canonical header bytes: `{"alg":"HS256","typ":"JWT"}` and `{"alg":"ES256","typ":"JWT"}`.
- 13 tests in new `SignerTests` suite.

**Out of scope (deferred to v0.3+):**
- RS256 (RSA-SHA-256), ES384, ES512, EdDSA (Ed25519).
- Header customization (`kid`, `jku`, `x5u`, `cty`).
- Claim builder helpers / JSON encoder integration.
- JWE (JSON Web Encryption).
- JWK / JWKS handling.
- Package rename `swift-jwt-verify` → `swift-jwt`.

**Anti-goals:**
- No new dependencies (swift-crypto already in v0.1).
- No new error cases (reuse `JWTVerifyError.invalidKey`).
- No changes to v0.1 verify APIs.
- No Foundation in `Sources/JWTVerify` (test target inherits same rule).

---

## Module structure

```
Sources/JWTVerify/
  Base64URL.swift              ← extend with encode(_:)
  JWTVerify.swift              ← add JWTSigner struct alongside JWTVerifier
  JWTAlgorithm.swift           ← unchanged
  JWTVerifyError.swift         ← unchanged
  Verifier.swift               ← unchanged
  Signer.swift                 ← NEW (~70 LOC)
  Documentation.docc/
    JWTVerify.md               ← v0.2 example added
Tests/JWTVerifyTests/
  SignerTests.swift            ← NEW (13 tests)
```

Total new source: ~140 LOC. Tests: ~250 LOC. Docs: ~50 LOC.

---

## Public API

```swift
public struct JWTSigner: Sendable {
    public let algorithm: JWTAlgorithm
    /// Algorithm-dependent key bytes:
    /// - `.hs256`: arbitrary-length HMAC secret bytes.
    /// - `.es256`: 32-byte P-256 private key in raw scalar form
    ///   (matches swift-crypto's `P256.Signing.PrivateKey(rawRepresentation:)`).
    public let key: [UInt8]

    public init(algorithm: JWTAlgorithm, key: [UInt8])

    /// Sign a payload as a JWS-compact JWT.
    /// Emits canonical header `{"alg":"...","typ":"JWT"}` and the algorithm-
    /// specific signature.
    ///
    /// - Throws: `JWTVerifyError.invalidKey` if `algorithm == .es256` and `key`
    ///   bytes don't parse as a valid P-256 private scalar.
    public func sign(_ payload: [UInt8]) throws(JWTVerifyError) -> String
}
```

Sits alongside the existing v0.1 `JWTVerifier` in `Sources/JWTVerify/JWTVerify.swift`.

### Key bytes asymmetry

| Algorithm | Verifier key | Signer key |
|---|---|---|
| `.hs256` | HMAC secret (any length) | HMAC secret (same; symmetric) |
| `.es256` | 65-byte P-256 public key (x963 uncompressed: `0x04 \|\| x \|\| y`) | **32-byte P-256 private key** (raw scalar) |

This is intentional and matches the underlying asymmetric algorithm. Adopters using ES256 typically obtain the pair via swift-crypto:

```swift
import Crypto

let priv = P256.Signing.PrivateKey()
let signer = JWTSigner(algorithm: .es256, key: Array(priv.rawRepresentation))
let verifier = JWTVerifier(algorithm: .es256, key: Array(priv.publicKey.x963Representation))
```

---

## Internals

### Base64URL.encode

In `Sources/JWTVerify/Base64URL.swift`, add to the existing `Base64URL` enum:

```swift
/// Encode bytes as a Base64URL string without padding (JWS-compact form).
static func encode(_ bytes: [UInt8]) -> String {
    Base64.encode(bytes, variant: .urlSafeNoPadding)
}
```

Stays internal (matches the existing `decode(_:)` visibility). Both functions live on the same `Base64URL` enum.

### Signer namespace

Create `Sources/JWTVerify/Signer.swift`:

```swift
// SPDX-License-Identifier: Apache-2.0 WITH LLVM-exception
// Copyright (c) 2026 The bare-swift Project Authors.

import Crypto
import Foundation  // for Data — required by swift-crypto signing APIs

/// Internal JWS-compact signing primitives. Mirror of ``Verifier``.
enum Signer {
    /// Canonical JWS header bytes for HS256. Field order `alg` then `typ`.
    static let headerHS256: [UInt8] = Array(#"{"alg":"HS256","typ":"JWT"}"#.utf8)

    /// Canonical JWS header bytes for ES256. Field order `alg` then `typ`.
    static let headerES256: [UInt8] = Array(#"{"alg":"ES256","typ":"JWT"}"#.utf8)

    /// HMAC-SHA-256 over `signingInput` with `key`. Total — no failures.
    static func signHS256(signingInput: [UInt8], key: [UInt8]) -> [UInt8] {
        let symmetric = SymmetricKey(data: key)
        var hmac = HMAC<SHA256>(key: symmetric)
        hmac.update(data: signingInput)
        return Array(hmac.finalize())
    }

    /// ECDSA P-256 over `signingInput` with a 32-byte raw private scalar.
    /// Returns the 64-byte R||S concatenated signature (RFC 7518 § 3.4).
    static func signES256(signingInput: [UInt8], privateKey: [UInt8]) throws(JWTVerifyError) -> [UInt8] {
        guard privateKey.count == 32 else { throw .invalidKey }
        let pk: P256.Signing.PrivateKey
        do {
            pk = try P256.Signing.PrivateKey(rawRepresentation: privateKey)
        } catch {
            throw .invalidKey
        }
        do {
            let sig = try pk.signature(for: Data(signingInput))
            return Array(sig.rawRepresentation)
        } catch {
            throw .invalidKey
        }
    }
}
```

**Note on Foundation:** the internal `Signer` file imports `Foundation` because `P256.Signing.PrivateKey.signature(for:)` accepts `Data`/`some DataProtocol` and the cleanest call site uses `Data(signingInput)`. Foundation is **not** exposed in the public API of `JWTVerify`. v0.1's `Verifier.swift` already imports Foundation for the same reason; this matches established convention.

### JWTSigner struct (added to JWTVerify.swift)

Append to `Sources/JWTVerify/JWTVerify.swift`:

```swift
/// JWS-compact JWT signer. Sendable + value-type; construct once and reuse.
///
/// Mirrors ``JWTVerifier``'s shape — `(algorithm, key)` config + a single
/// operation function. The `key` bytes are algorithm-dependent (see
/// individual algorithm docs).
public struct JWTSigner: Sendable {
    public let algorithm: JWTAlgorithm
    /// Algorithm-dependent key bytes:
    /// - `.hs256`: arbitrary-length HMAC secret bytes.
    /// - `.es256`: 32-byte P-256 private key in raw scalar form
    ///   (matches swift-crypto's `P256.Signing.PrivateKey(rawRepresentation:)`).
    public let key: [UInt8]

    public init(algorithm: JWTAlgorithm, key: [UInt8]) {
        self.algorithm = algorithm
        self.key = key
    }

    /// Sign a payload as a JWS-compact JWT.
    public func sign(_ payload: [UInt8]) throws(JWTVerifyError) -> String {
        let headerBytes: [UInt8]
        switch algorithm {
        case .hs256: headerBytes = Signer.headerHS256
        case .es256: headerBytes = Signer.headerES256
        }
        let h = Base64URL.encode(headerBytes)
        let p = Base64URL.encode(payload)
        let signingInput = Array((h + "." + p).utf8)

        let signature: [UInt8]
        switch algorithm {
        case .hs256:
            signature = Signer.signHS256(signingInput: signingInput, key: key)
        case .es256:
            signature = try Signer.signES256(signingInput: signingInput, privateKey: key)
        }
        let s = Base64URL.encode(signature)
        return h + "." + p + "." + s
    }
}
```

---

## Test plan (13 tests, 1 suite: `SignerTests`)

All Foundation-free at the test level; swift-crypto types appear only inside test helpers that generate ES256 key pairs.

1. **HS256 deterministic vector.** Sign a known payload (`{"sub":"alice"}`) with a known 32-byte secret. Assert the resulting token matches an externally-computed expected token byte-for-byte (HS256 is deterministic; we compute the expected token once via a Python `jose` script and bake it into the test as a literal).

2. **HS256 round-trip.** `JWTSigner.sign` → `JWTVerifier.verify` succeeds; `verified.payloadBytes` matches input.

3. **HS256 multi-byte secret round-trip.** 64-byte HMAC secret round-trips.

4. **HS256 empty payload round-trip.** Empty `[]` payload round-trips through verify.

5. **ES256 round-trip with generated key pair.**
   ```swift
   let priv = P256.Signing.PrivateKey()
   let signer = JWTSigner(algorithm: .es256, key: Array(priv.rawRepresentation))
   let verifier = JWTVerifier(algorithm: .es256, key: Array(priv.publicKey.x963Representation))
   let token = try signer.sign(payload)
   let verified = try verifier.verify(token)
   #expect(verified.payloadBytes == payload)
   ```

6. **ES256 wrong private key length** (33 bytes) throws `JWTVerifyError.invalidKey`.

7. **ES256 zero private scalar** (32 bytes of `0x00`) throws `JWTVerifyError.invalidKey`.

8. **JWS header byte sequence is canonical.** Sign with HS256; split token at first `.`; base64URL-decode the header segment; assert exactly `{"alg":"HS256","typ":"JWT"}`.

9. **Sign produces 3-segment token.** Assert token contains exactly 2 dots.

10. **HS256 + ES256 produce different tokens for same payload** (different header → different signing input → different signature; ES256 also has different signature shape).

11. **HS256 signature length is 32 bytes.** Base64URL-decode the signature segment; assert `.count == 32`.

12. **ES256 signature length is 64 bytes.** Base64URL-decode the signature segment; assert `.count == 64`.

13. **Base64URL output uses URL-safe alphabet.** Sign a payload chosen to produce `+` / `/` in standard base64 (e.g., `[0xfb, 0xff, 0xbf]`); assert the resulting token contains neither `+` nor `/` nor `=`.

---

## CHANGELOG entry

```markdown
## [0.2.0] - 2026-05-16

### Added
- `JWTSigner` struct — Sendable + value-type JWS-compact signer mirroring `JWTVerifier`'s shape. `(algorithm: JWTAlgorithm, key: [UInt8])` config; `sign(_:) throws(JWTVerifyError) -> String` produces a JWS-compact JWT.
- HS256 signing via swift-crypto's `HMAC<SHA256>`. Symmetric — same key bytes as v0.1's HS256 verifier.
- ES256 signing via swift-crypto's `P256.Signing.PrivateKey`. **Key format: 32-byte raw private scalar** (matches `P256.Signing.PrivateKey(rawRepresentation:)`); the matching ES256 verifier still uses 65-byte x963 uncompressed public key (`0x04 || x || y`). Adopters derive the pair via `priv.rawRepresentation` (signer) + `priv.publicKey.x963Representation` (verifier).
- Internal `Base64URL.encode(_:)` for the JWS-compact (URL-safe, unpadded) format.
- Canonical JWS header bytes: `{"alg":"HS256","typ":"JWT"}` and `{"alg":"ES256","typ":"JWT"}`.
- 13 new tests in `SignerTests` suite: HS256 deterministic vector, round-trips (HS256 + ES256 with various inputs), signature length assertions, URL-safe alphabet check, header byte-canonicality, error cases.

### Changed
- README + DocC clarify that v0.2+ supports both signing and verification despite the package name `swift-jwt-verify`.

### Dependencies
- No new dependencies. swift-crypto + swift-base64 already in v0.1.

### Migration (v0.1 → v0.2)
- **Additive only — non-breaking.** All v0.1 `JWTVerifier` APIs unchanged. v0.2 only adds the new `JWTSigner` struct.

### Phase 19
- Tranche 19A of [RFC-0024](https://github.com/bare-swift/bare-swift/blob/main/rfcs/0024-phase-19-anchor-swift-jwt-verify-v0.2-signing.md). Closes the longest-standing functional gap in the auth tier — the verify-only design boundary deferred for 7 consecutive gates.
```

---

## README updates

- Bump install snippet `from: "0.1.0"` → `from: "0.2.0"`.
- Update tagline / overview to note signing is now available.
- Add a `## Signing (v0.2+)` section after the existing verification example with an HS256 sign+verify round-trip and the ES256 key pair conversion idiom.
- Note that the package name `swift-jwt-verify` is preserved despite gaining sign capability; package rename is a Phase 20+ consideration.

---

## DocC updates

In `Sources/JWTVerify/Documentation.docc/JWTVerify.md`:

- Add a brief signing example in the Overview.
- Add `JWTSigner` to the Topics list under a new "Signing" subsection.
- No cross-package symbol links (per [[feedback-docc-cross-package]]).

---

## CI considerations

No CI workflow changes. Existing reusable workflow handles the build/test/sanitizers/docc matrix. Sanitizers stay ON (no large static data).

---

## Ship sequencing

1. Add `Base64URL.encode(_:)` in `Base64URL.swift`.
2. Create `Sources/JWTVerify/Signer.swift` with internal namespace.
3. Add public `JWTSigner` struct to `JWTVerify.swift`.
4. Write `Tests/JWTVerifyTests/SignerTests.swift` (13 tests + key-pair helpers).
5. Run full package suite (v0.1 verify tests + new signer tests) — confirm green.
6. Update CHANGELOG + README + DocC catalog.
7. Build DocC with warnings-as-errors locally.
8. Commit, push, watch CI.
9. Tag v0.2.0; watch Release + Publish docs.
10. Umbrella bump (`swift-jwt-verify` 0.1.0 → 0.2.0 in `packages/index.json`).
11. Memory closure (`project_phase19_19a_shipped.md`).

Total Phase 19 calendar: ~0.5-1 day wall-clock (matches RFC-0024 Shape A).

---

## Decisions log (locked)

- **Shape A** (single tranche, HS256 + ES256 in one ship).
- **Module name stays `JWTVerify`**, package name stays `swift-jwt-verify`. Despite the verify-only naming, v0.2 adds signing under the same module. Rename deferred to Phase 20+.
- **`JWTSigner` struct shape mirrors `JWTVerifier`** (non-throwing init; throwing operation; `(algorithm, key)` config).
- **ES256 key format**: 32-byte raw scalar (swift-crypto's `P256.Signing.PrivateKey(rawRepresentation:)`). Adopters convert from the private key's `rawRepresentation` for signing and `.publicKey.x963Representation` for verification.
- **HS256 key**: arbitrary length (no minimum enforced; matches v0.1 verifier).
- **Canonical header order**: `alg` then `typ`. Hardcoded byte sequences in `Signer`.
- **Signing error path**: reuse `JWTVerifyError.invalidKey`. No new error cases.
- **Foundation in `Signer.swift`** (internal only) for `Data(signingInput)` argument to swift-crypto's signing API. Public API stays Foundation-free. Matches v0.1's `Verifier.swift` convention.
- **HS256 has deterministic vector test**; ES256 only round-trip (signatures randomized by k).
- **Base64URL.encode wraps `.urlSafeNoPadding`** variant of swift-base64. Stays `internal`.
- **13 tests in single suite** (`SignerTests`) — shared fixtures.
- **No claim builder, no header customization, no JWE, no rename.** Phase 20+ candidates.

# swift-jwt-verify v0.3 (ES384 + ES512 + EdDSA + `kid` header) Design

**Phase 21 Tranche 21A** (RFC-0026). Existing package; additive non-breaking minor version bump.

**Date:** 2026-05-16

---

## Goal

Add ES384, ES512, and EdDSA (Ed25519) sign+verify primitives matching v0.2's `Signer`/`Verifier` pattern exactly, plus optional `kid` header customization on `JWTSigner`. Closes the documented algorithm-completion deferral for ECDSA + EdDSA families (RS-family deferred to v0.4 pending swift-crypto RSA SPI stabilization). Non-breaking — all v0.2 verify+sign APIs unchanged.

## Scope (Shape A — single tranche, 3 algorithms + `kid`)

**In scope:**
- 3 new `JWTAlgorithm` enum cases: `.es384`, `.es512`, `.eddsa`.
- `JWTAlgorithm.rfc7518Name` returns `"ES384"` / `"ES512"` / `"EdDSA"`.
- `JWTAlgorithm.fromName(_:)` recognizes the new names.
- 3 new `Signer.sign*` static functions (mirror `signES256` pattern).
- 3 new `Verifier.verify*` static functions (mirror `verifyES256` pattern).
- 3 new canonical header byte constants.
- `JWTSigner.kid: String?` optional init parameter (default nil; v0.2 behavior preserved).
- Dynamic header builder when `kid != nil`; backslash + double-quote escaping.
- ~25 new tests across 3 existing suites.

**Out of scope (deferred to v0.4+):**
- RS256 / RS384 / RS512 / PS256 (swift-crypto `_RSA` experimental SPI).
- `jku` / `x5u` / `cty` / `crit` / non-`JWT` `typ` headers.
- JWK / JWKS endpoint fetching.
- JWE.
- Claim builder helpers / JSON encoder integration.
- Package rename `swift-jwt-verify` → `swift-jwt`.

**Anti-goals:**
- No changes to v0.2 public API surface.
- No new dependencies (swift-crypto + swift-base64 already in v0.1).
- No new error cases — reuse `JWTVerifyError.invalidKey` and `.signatureMismatch`.
- Public API stays Foundation-free (matches v0.1/v0.2 pattern: internal `Signer.swift` + `Verifier.swift` import Foundation for swift-crypto `Data` bridging only).

---

## File changes

```
swift-jwt-verify/
  Package.swift                                ← unchanged
  Sources/JWTVerify/
    JWTAlgorithm.swift                         ← extend enum + rfc7518Name + fromName
    Signer.swift                               ← +3 header constants + 3 sign funcs (~70 LOC)
    Verifier.swift                             ← +3 verify funcs (~50 LOC)
    JWTVerify.swift                            ← extend dispatch switches + JWTSigner.kid + buildHeader helper
    Documentation.docc/JWTVerify.md            ← add v0.3 note + algorithm topic
  Tests/JWTVerifyTests/
    JWTAlgorithmTests.swift                    ← +3 tests (rfc7518Name + fromName for new algos)
    SignerTests.swift                          ← +12 tests (3 algos + kid emission)
    VerifierTests.swift                        ← +3 tests (round-trip per algo)
  CHANGELOG.md                                 ← v0.3.0 entry
  README.md                                    ← install version + algorithm note
```

Total new source: ~185 LOC. Tests: ~250 LOC. Docs: ~50 LOC.

---

## Public API additions

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

public struct JWTSigner: Sendable {
    public let algorithm: JWTAlgorithm
    public let key: [UInt8]
    public let kid: String?   // NEW (default nil)

    public init(algorithm: JWTAlgorithm, key: [UInt8], kid: String? = nil)

    public func sign(_ payload: [UInt8]) throws(JWTVerifyError) -> String
}
```

`JWTVerifier` signature unchanged. `JWTVerifyError` unchanged.

### Key bytes per algorithm (locked)

| Algorithm | Signer key | Verifier key | Signature size |
|---|---|---|---|
| `.hs256` | HMAC secret (any length) | Same (symmetric) | 32B |
| `.es256` | 32B P-256 raw scalar | 65B P-256 x963 | 64B R\|\|S |
| `.es384` | **48B P-384 raw scalar** | **97B P-384 x963** (`0x04 \|\| 48B x \|\| 48B y`) | **96B R\|\|S** |
| `.es512` | **66B P-521 raw scalar** | **133B P-521 x963** (`0x04 \|\| 66B x \|\| 66B y`) | **132B R\|\|S** |
| `.eddsa` | **32B Ed25519 seed** | **32B Ed25519 raw** (NOT x963) | **64B raw** |

EdDSA is notably different: symmetric key sizes (32B each), no x963 wrapper. Adopters use `priv.publicKey.rawRepresentation` (not `x963Representation`) for the matching verifier key.

---

## Internals

### `Signer` namespace additions (in `Sources/JWTVerify/Signer.swift`)

Append after the existing extension that holds `signHS256` / `signES256`:

```swift
extension Signer {
    static let headerES384: [UInt8] = Array(#"{"alg":"ES384","typ":"JWT"}"#.utf8)
    static let headerES512: [UInt8] = Array(#"{"alg":"ES512","typ":"JWT"}"#.utf8)
    static let headerEdDSA: [UInt8] = Array(#"{"alg":"EdDSA","typ":"JWT"}"#.utf8)

    static func signES384(signingInput: [UInt8], privateKey: [UInt8]) throws(JWTVerifyError) -> [UInt8] {
        guard privateKey.count == 48 else { throw .invalidKey }
        let pk: P384.Signing.PrivateKey
        do { pk = try P384.Signing.PrivateKey(rawRepresentation: privateKey) }
        catch { throw .invalidKey }
        do {
            let sig = try pk.signature(for: Data(signingInput))
            return Array(sig.rawRepresentation)
        } catch { throw .invalidKey }
    }

    static func signES512(signingInput: [UInt8], privateKey: [UInt8]) throws(JWTVerifyError) -> [UInt8] {
        guard privateKey.count == 66 else { throw .invalidKey }
        let pk: P521.Signing.PrivateKey
        do { pk = try P521.Signing.PrivateKey(rawRepresentation: privateKey) }
        catch { throw .invalidKey }
        do {
            let sig = try pk.signature(for: Data(signingInput))
            return Array(sig.rawRepresentation)
        } catch { throw .invalidKey }
    }

    /// EdDSA / Ed25519 signing. `privateKey` is the 32-byte seed.
    /// Returns the raw 64-byte signature.
    static func signEdDSA(signingInput: [UInt8], privateKey: [UInt8]) throws(JWTVerifyError) -> [UInt8] {
        guard privateKey.count == 32 else { throw .invalidKey }
        let pk: Curve25519.Signing.PrivateKey
        do { pk = try Curve25519.Signing.PrivateKey(rawRepresentation: privateKey) }
        catch { throw .invalidKey }
        do {
            let sig = try pk.signature(for: Data(signingInput))
            return Array(sig)   // swift-crypto returns Data directly for Curve25519
        } catch { throw .invalidKey }
    }
}
```

### `Verifier` namespace additions (in `Sources/JWTVerify/Verifier.swift`)

Append after the existing `verifyES256` extension:

```swift
extension Verifier {
    static func verifyES384(signingInput: [UInt8], signature: [UInt8], publicKey: [UInt8]) throws(JWTVerifyError) {
        guard signature.count == 96 else { throw .signatureMismatch }
        guard publicKey.count == 97, publicKey[0] == 0x04 else { throw .invalidKey }
        let pk: P384.Signing.PublicKey
        do { pk = try P384.Signing.PublicKey(x963Representation: publicKey) }
        catch { throw .invalidKey }
        let sig: P384.Signing.ECDSASignature
        do { sig = try P384.Signing.ECDSASignature(rawRepresentation: signature) }
        catch { throw .signatureMismatch }
        if !pk.isValidSignature(sig, for: Data(signingInput)) { throw .signatureMismatch }
    }

    static func verifyES512(signingInput: [UInt8], signature: [UInt8], publicKey: [UInt8]) throws(JWTVerifyError) {
        guard signature.count == 132 else { throw .signatureMismatch }
        guard publicKey.count == 133, publicKey[0] == 0x04 else { throw .invalidKey }
        let pk: P521.Signing.PublicKey
        do { pk = try P521.Signing.PublicKey(x963Representation: publicKey) }
        catch { throw .invalidKey }
        let sig: P521.Signing.ECDSASignature
        do { sig = try P521.Signing.ECDSASignature(rawRepresentation: signature) }
        catch { throw .signatureMismatch }
        if !pk.isValidSignature(sig, for: Data(signingInput)) { throw .signatureMismatch }
    }

    /// EdDSA / Ed25519 verification. `publicKey` is the 32-byte raw public key
    /// (NOT x963 — Ed25519 has no x963 form).
    static func verifyEdDSA(signingInput: [UInt8], signature: [UInt8], publicKey: [UInt8]) throws(JWTVerifyError) {
        guard signature.count == 64 else { throw .signatureMismatch }
        guard publicKey.count == 32 else { throw .invalidKey }
        let pk: Curve25519.Signing.PublicKey
        do { pk = try Curve25519.Signing.PublicKey(rawRepresentation: publicKey) }
        catch { throw .invalidKey }
        if !pk.isValidSignature(Data(signature), for: Data(signingInput)) {
            throw .signatureMismatch
        }
    }
}
```

### `JWTSigner.sign` dispatch + `kid` wiring + buildHeader helper

In `Sources/JWTVerify/JWTVerify.swift`, replace the existing `JWTSigner` struct's body (`init` + `sign`) with:

```swift
public struct JWTSigner: Sendable {
    public let algorithm: JWTAlgorithm
    public let key: [UInt8]
    public let kid: String?

    public init(algorithm: JWTAlgorithm, key: [UInt8], kid: String? = nil) {
        self.algorithm = algorithm
        self.key = key
        self.kid = kid
    }

    public func sign(_ payload: [UInt8]) throws(JWTVerifyError) -> String {
        let headerBytes: [UInt8]
        if let kid {
            headerBytes = JWTSigner.buildHeader(algorithm: algorithm.rfc7518Name, kid: kid)
        } else {
            switch algorithm {
            case .hs256: headerBytes = Signer.headerHS256
            case .es256: headerBytes = Signer.headerES256
            case .es384: headerBytes = Signer.headerES384
            case .es512: headerBytes = Signer.headerES512
            case .eddsa: headerBytes = Signer.headerEdDSA
            }
        }
        let h = Base64URL.encode(headerBytes)
        let p = Base64URL.encode(payload)
        let signingInput = Array((h + "." + p).utf8)

        let signature: [UInt8]
        switch algorithm {
        case .hs256: signature = Signer.signHS256(signingInput: signingInput, key: key)
        case .es256: signature = try Signer.signES256(signingInput: signingInput, privateKey: key)
        case .es384: signature = try Signer.signES384(signingInput: signingInput, privateKey: key)
        case .es512: signature = try Signer.signES512(signingInput: signingInput, privateKey: key)
        case .eddsa: signature = try Signer.signEdDSA(signingInput: signingInput, privateKey: key)
        }
        let s = Base64URL.encode(signature)
        return h + "." + p + "." + s
    }

    /// Build dynamic JWS header bytes with optional `kid` field. Escapes
    /// `"` and `\` in the kid value per JSON spec; assumes other characters
    /// pass through (typical kid values are alphanumerics + `-_.:` which
    /// don't need escaping in JSON strings).
    @usableFromInline
    internal static func buildHeader(algorithm: String, kid: String?) -> [UInt8] {
        var out = Array(#"{"alg":""#.utf8)
        out.append(contentsOf: algorithm.utf8)
        out.append(contentsOf: Array(#"","typ":"JWT""#.utf8))
        if let kid {
            out.append(contentsOf: Array(#","kid":""#.utf8))
            for ch in kid.utf8 {
                if ch == 0x22 || ch == 0x5C {   // escape " or \
                    out.append(0x5C)
                }
                out.append(ch)
            }
            out.append(0x22)   // closing "
        }
        out.append(0x7D)   // }
        return out
    }
}
```

### `JWTVerifier.verify` dispatch

In `Sources/JWTVerify/JWTVerify.swift`, extend the existing switch in `JWTVerifier.verify(_:)`:

```swift
switch algorithm {
case .hs256:
    try Verifier.verifyHS256(signingInput: parts.signingInput, signature: signature, key: key)
case .es256:
    try Verifier.verifyES256(signingInput: parts.signingInput, signature: signature, publicKey: key)
case .es384:
    try Verifier.verifyES384(signingInput: parts.signingInput, signature: signature, publicKey: key)
case .es512:
    try Verifier.verifyES512(signingInput: parts.signingInput, signature: signature, publicKey: key)
case .eddsa:
    try Verifier.verifyEdDSA(signingInput: parts.signingInput, signature: signature, publicKey: key)
}
```

---

## Test plan (~25 new tests, 3 existing suites extended)

### JWTAlgorithmTests (3 new — in `Tests/JWTVerifyTests/JWTAlgorithmTests.swift`)

1. `JWTAlgorithm.es384.rfc7518Name == "ES384"`.
2. `JWTAlgorithm.es512.rfc7518Name == "ES512"`.
3. `JWTAlgorithm.eddsa.rfc7518Name == "EdDSA"` + `JWTAlgorithm.fromName(_:)` round-trip for all 3.

### SignerTests (12 new — in `Tests/JWTVerifyTests/SignerTests.swift`)

ES384:
1. ES384 round-trip (sign → verify with matching key pair).
2. ES384 wrong-length key (47B) throws `.invalidKey`.
3. ES384 signature segment decodes to 96B.

ES512:
4. ES512 round-trip.
5. ES512 wrong-length key (65B) throws `.invalidKey`.
6. ES512 signature segment decodes to 132B.

EdDSA:
7. EdDSA round-trip.
8. EdDSA wrong-length key (31B) throws `.invalidKey`.
9. EdDSA signature segment decodes to 64B.
10. EdDSA key symmetric sizes (private + public both 32B).

`kid` header:
11. `kid: nil` produces v0.2 canonical header byte-for-byte (regression check).
12. `kid: "key-1"` produces header `{"alg":"HS256","typ":"JWT","kid":"key-1"}` (decode + assert exact string).
13. `kid: "has\"quote"` escapes the quote in header output.

(12 + 1 baseline-canonical = 13; spec said "~12-15", landing at 13.)

### VerifierTests (3 new — in `Tests/JWTVerifyTests/VerifierTests.swift`)

Verify round-trips: caller signs with `JWTSigner` and verifies with `JWTVerifier` for each new algorithm. Both invocation paths exercise the JWTVerifier.verify dispatch switch.

1. ES384 round-trip through `JWTVerifier.verify(_:)`.
2. ES512 round-trip through `JWTVerifier.verify(_:)`.
3. EdDSA round-trip through `JWTVerifier.verify(_:)`.

All tests Foundation-free at the test level; swift-crypto types appear only inside test helpers that generate key pairs.

---

## CHANGELOG entry

```markdown
## [0.3.0] - 2026-05-16

### Added
- **3 new JWT algorithms** for sign + verify:
  - `JWTAlgorithm.es384` — ECDSA P-384 + SHA-384 (RFC 7518 § 3.4). 48-byte private scalar; 97-byte x963 public key.
  - `JWTAlgorithm.es512` — ECDSA P-521 + SHA-512 (RFC 7518 § 3.4). 66-byte private scalar; 133-byte x963 public key.
  - `JWTAlgorithm.eddsa` — EdDSA via Ed25519 (RFC 8037). 32-byte seed; 32-byte raw public key (NOT x963).
- **Optional `kid` header customization** on `JWTSigner`. `JWTSigner(algorithm:, key:, kid:)` — when `kid != nil`, the emitted JWS header includes `"kid":"<value>"`. v0.2 callers continue to get the canonical static header `{"alg":"...","typ":"JWT"}` byte-for-byte.
- ~25 new tests covering all three algorithms (round-trip + error cases + signature length) and `kid` emission (nil canonical regression + non-nil dynamic header + escaping).

### Dependencies
- No new dependencies. swift-crypto + swift-base64 already in v0.1.

### Key bytes (new algorithms)
| Algorithm | Signer key | Verifier key | Signature |
|---|---|---|---|
| `.es384` | 48B P-384 raw scalar | 97B P-384 x963 (`0x04 || x || y`) | 96B R||S |
| `.es512` | 66B P-521 raw scalar | 133B P-521 x963 | 132B R||S |
| `.eddsa` | 32B Ed25519 seed | **32B Ed25519 raw** (no x963 wrapper) | 64B raw |

Adopters derive each pair via swift-crypto:
- ES384/ES512: `priv.rawRepresentation` (signer) + `priv.publicKey.x963Representation` (verifier).
- EdDSA: `priv.rawRepresentation` (signer) + `priv.publicKey.rawRepresentation` (verifier).

### Migration (v0.2 → v0.3)
- **Additive only — non-breaking.** All v0.2 APIs unchanged.
- `JWTSigner(algorithm:, key:)` call sites continue to work; `kid` has a default of nil.
- `JWTVerifier` API surface unchanged.

### Out of scope (deferred to v0.4+)
- RS256 / RS384 / RS512 / PS256 (swift-crypto's RSA API still under `_RSA` experimental SPI).
- `jku` / `x5u` / `cty` headers.
- JWE / JWK / JWKS.
- Claim builder helpers.
- Package rename `swift-jwt-verify` → `swift-jwt`.

### Phase 21
- Tranche 21A of [RFC-0026](https://github.com/bare-swift/bare-swift/blob/main/rfcs/0026-phase-21-anchor-swift-jwt-verify-v0.3-algorithm-expansion.md). Closes the algorithm-completion deferral for ECDSA + EdDSA families.
```

---

## README updates

- Bump install snippet `from: "0.2.0"` → `from: "0.3.0"`.
- Update tagline: "JWS-compact JWT verifier and signer (HS256 + ES256 + ES384 + ES512 + EdDSA, with optional `kid`)".
- Add ES384/ES512/EdDSA signing examples (similar shape to the v0.2 HS256 + ES256 examples).
- Add a `### `kid` header (v0.3+)` subsection demonstrating multi-key verifier scenarios.
- Update the "Supported algorithms" section.
- Update "Out of scope" to defer RS-family and jku/x5u/cty to v0.4+.

---

## DocC catalog updates

In `Sources/JWTVerify/Documentation.docc/JWTVerify.md`:

- Update Overview to mention v0.3 ships 5 algorithms total.
- Update the existing topic group structure to keep `JWTSigner` / `JWTVerifier` / `JWTAlgorithm` listed.
- Add `### Header customization (v0.3+)` topic group with prose pointing at the README for `kid` usage examples.
- No cross-package symbol links.

---

## CI considerations

No CI workflow changes. Existing reusable workflow handles the build/test/sanitizers/docc matrix. Sanitizers stay ON (no large static data).

---

## Migration

**Additive only — non-breaking.** v0.2 callers' code compiles and behaves identically against v0.3:
- `JWTSigner(algorithm: .hs256, key: secret)` works (kid defaults nil → v0.2 canonical header).
- `JWTSigner(algorithm: .es256, key: privKey32)` works.
- `JWTVerifier(algorithm: .hs256, key: secret).verify(token)` works.

Adopters add new algorithms by passing `.es384` / `.es512` / `.eddsa` with the appropriately-sized key bytes per the table above.

---

## Ship sequencing

1. Extend `JWTAlgorithm.swift` (3 cases + rfc7518Name + fromName).
2. Append to `Signer.swift` (3 header constants + 3 sign functions).
3. Append to `Verifier.swift` (3 verify functions).
4. Rewrite `JWTSigner` body in `JWTVerify.swift` (kid field + dispatch switch + buildHeader helper) + extend `JWTVerifier.verify` switch.
5. Extend `JWTAlgorithmTests` + `SignerTests` + `VerifierTests`.
6. Run full package suite (33 v0.2 tests + 25 new tests = ~58 tests).
7. Update CHANGELOG + README + DocC.
8. Build DocC locally with `--warnings-as-errors`.
9. Commit, push, watch CI.
10. Tag v0.3.0; watch Release + Publish docs.
11. Umbrella bump (`swift-jwt-verify` 0.2.0 → 0.3.0 in `packages/index.json`).
12. Memory closure (`project_phase21_21a_shipped.md`).

Total Phase 21 calendar: ~1-2 hours wall-clock per Gate 20's calibration.

---

## Decisions log (locked)

- **3 algorithms in one ship** (ES384 + ES512 + EdDSA). RS-family deferred to v0.4 (swift-crypto `_RSA` SPI).
- **`kid` only** for v0.3 headers; `jku` / `x5u` / `cty` v0.4+.
- **EdDSA public key format = raw 32B** (NOT x963 wrapped). Differs from ES* family; documented in CHANGELOG + README.
- **`kid` escape: only `"` and `\`**. JSON spec requires escaping these two; common `kid` values (alphanumerics + `-_.:` URI characters) don't need any other escaping. Documented as caveat.
- **Hand-rolled `buildHeader`** for `kid` (no JSON encoder dep; matches v0.1's `extractAlg` minimal-JSON pattern).
- **`JWTSigner.kid: String?` default nil** — preserves v0.2 byte-for-byte header output when nil; regression-tested.
- **Reuse `JWTVerifyError.invalidKey` and `.signatureMismatch`** — no new error cases.
- **Foundation-in-internal-only** pattern preserved (Signer.swift + Verifier.swift import Foundation for swift-crypto Data bridging).
- **Module name `JWTVerify`** and package name `swift-jwt-verify` preserved (rename deferred to Phase 23+).
- **JWTVerifier API surface unchanged** — verify-side gets new algorithms via dispatch switch only.
- **Test count ~25 new** across 3 existing test files (no new test files).

# swift-oauth2-client v0.2 (auth flow start) — Design

**Phase 26 Tranche 26A** (RFC-0031). Existing package; additive non-breaking minor version bump.

**Date:** 2026-05-16

## Goal

Add the "auth flow start" features deferred from Phase 20 v0.1: authorization URL builder, PKCE primitives (verifier + challenge), state/nonce random token generator, and `ClientAuthMethod` enum (with HTTP Basic auth support). v0.1 + v0.2 together cover the complete OAuth 2.0 client surface (token endpoint + auth flow start).

## Reality-check findings

- **swift-crypto** needed only for `SHA256.hash(data:)` (PKCE S256 challenge). Stable API.
- **Secure random** via Swift stdlib's `SystemRandomNumberGenerator` (CSPRNG on macOS and Linux). swift-crypto not needed for random bytes.
- **swift-base64 dep avoided** — v0.1's hand-rolled-pattern continues. Inline ~50 LOC base64 helper supporting both standard (with `=` padding, `+/`) and url (no padding, `-_`) variants.
- **`FormEncoder` reusable** for query-string emit per RFC 6749 § 3.1 (auth URL params use form-urlencoding).
- **RFC 6749 § 2.3.1 HTTP Basic for OAuth** requires URL-encoding `client_id` + `client_secret` BEFORE base64-encoding the `id:secret` pair. Different from plain RFC 7617 Basic. Codified in design.
- v0.1's existing exhaustive switch over `OAuth2ClientError` at `OAuth2ClientTests.swift:` is at the API-shape tests file. May need update if new error cases are added.
- `OAuth2Client.swift` has 3 fields currently — adding `clientAuthMethod: ClientAuthMethod = .body` is additive (default preserves v0.1).

**Conclusion:** Single tranche, all four v0.2 features. Estimated ~1.5-2 hours per RFC-0031's bracket (NOT a wrapper-pattern package — ships net-new features).

## Scope

**In scope:**
- `ClientAuthMethod` enum: `.body` (v0.1 behavior; default) / `.basic` (HTTP Basic per RFC 6749 § 2.3.1).
- `OAuth2Client.clientAuthMethod` field + new init param (default `.body`).
- `OAuth2Client.basicAuthHeader() -> (name: String, value: String)?` — returns `("Authorization", "Basic <encoded>")` when `.basic`, nil when `.body` or when `clientSecret == nil`.
- `OAuth2Client.requestBody(grant:)` updated: when `.basic`, omits `client_id` + `client_secret` from body.
- `OAuth2Client.authorizationURL(...) -> String` — builds RFC 6749 § 4.1.1 authorization URL with query params.
- `OAuth2Client.PKCE` namespace enum:
  - `PKCE.generateVerifier(byteCount: Int = 32) -> String` — random URL-safe verifier (RFC 7636 § 4.1; 43-char default for 32 bytes).
  - `PKCE.challenge(for verifier: String, method: PKCEMethod) -> String` — RFC 7636 § 4.2 challenge (plain or S256).
- `PKCEMethod` enum: `.plain`, `.s256`. `var rfc7636Name: String` (`"plain"` / `"S256"`).
- `OAuth2Client.randomToken(byteCount: Int = 16) -> String` — URL-safe random token (for state/nonce).
- Internal inline base64 helper (~50 LOC) supporting standard + url variants.
- ~25 new tests.

**Out of scope (deferred to v0.3+):**
- Token caching / lifecycle / auto-refresh.
- OIDC ID-token helpers (composes with swift-jwt-verify).
- Implicit / password / device-code grants.
- mTLS / `private_key_jwt` client authentication.
- JWT-based `client_assertion`.
- HTTP transport.
- AuthorizationURL response-type override beyond "code" (implicit/hybrid grants explicitly out).

**Anti-goals:**
- No changes to v0.1 byte-for-byte output for `.body` auth method (regression test).
- No swift-base64 dep (inline base64 helper).
- No swift-uri dep (string concat for URL building).
- No URL type (return String).

## File changes

```
swift-oauth2-client/
  Package.swift                              ← add swift-crypto dep
  Sources/OAuth2Client/
    OAuth2Client.swift                       ← extend with clientAuthMethod + basicAuthHeader + authorizationURL + randomToken (~80 LOC change)
    OAuth2ClientError.swift                  ← unchanged (no new error cases needed; all impossible inputs are caller-side bugs)
    ClientAuthMethod.swift                   ← NEW: ClientAuthMethod enum (~25 LOC)
    PKCE.swift                               ← NEW: PKCEMethod enum + OAuth2Client.PKCE namespace (~100 LOC)
    Base64.swift                             ← NEW: internal base64 helper (standard + url) (~80 LOC)
    Documentation.docc/OAuth2Client.md       ← Auth flow topic group (~30 LOC)
  Tests/OAuth2ClientTests/
    OAuth2ClientTests.swift                  ← extend with ClientAuthMethod regression (~5 LOC)
    AuthorizationURLTests.swift              ← NEW: ~10 tests (~150 LOC)
    PKCETests.swift                          ← NEW: ~6 tests including RFC 7636 § 4.2 worked example (~100 LOC)
    ClientAuthMethodTests.swift              ← NEW: ~5 tests for .basic flow (~80 LOC)
    Base64Tests.swift                        ← NEW: ~5 tests for inline helper (~70 LOC)
  CHANGELOG.md                               ← v0.2.0 entry
  README.md                                  ← v0.2 install + auth flow examples
```

## Public API additions

```swift
public enum ClientAuthMethod: Sendable, Equatable {
    case body                   // RFC 6749 § 2.3.1 in-body credentials (v0.1 default).
    case basic                  // RFC 6749 § 2.3.1 HTTP Basic header.
}

public enum PKCEMethod: Sendable, Equatable {
    case plain                  // RFC 7636 § 4.2 — verifier = challenge.
    case s256                   // RFC 7636 § 4.2 — base64url(SHA256(verifier)).

    public var rfc7636Name: String {
        self == .s256 ? "S256" : "plain"
    }
}

public struct OAuth2Client: Sendable {
    public let tokenEndpoint: String
    public let clientID: String
    public let clientSecret: String?
    public let clientAuthMethod: ClientAuthMethod   // NEW

    public init(
        tokenEndpoint: String,
        clientID: String,
        clientSecret: String? = nil,
        clientAuthMethod: ClientAuthMethod = .body    // NEW; default preserves v0.1
    )

    public func requestBody(grant: GrantType) -> Bytes
    public func basicAuthHeader() -> (name: String, value: String)?   // NEW
    public func parseResponse(_ body: Bytes) throws(OAuth2ClientError) -> TokenResponse

    public func authorizationURL(                                       // NEW
        authorizationEndpoint: String,
        redirectURI: String,
        scope: String? = nil,
        state: String? = nil,
        nonce: String? = nil,
        codeChallenge: String? = nil,
        codeChallengeMethod: PKCEMethod? = nil,
        responseType: String = "code",
        additionalParams: [(name: String, value: String)] = []
    ) -> String

    public enum PKCE: Sendable {                                        // NEW
        public static func generateVerifier(byteCount: Int = 32) -> String
        public static func challenge(for verifier: String, method: PKCEMethod) -> String
    }

    public static func randomToken(byteCount: Int = 16) -> String       // NEW
}
```

## Internals

### `Sources/OAuth2Client/ClientAuthMethod.swift` (NEW)

```swift
// SPDX-License-Identifier: Apache-2.0 WITH LLVM-exception
// Copyright (c) 2026 The bare-swift Project Authors.

/// How client credentials (`client_id` + `client_secret`) are conveyed to the
/// token endpoint per RFC 6749 § 2.3.1.
public enum ClientAuthMethod: Sendable, Equatable {
    /// Credentials in the form-encoded request body (v0.1 default).
    /// `requestBody(grant:)` includes `client_id` (and `client_secret` if set).
    case body

    /// Credentials in an `Authorization: Basic` HTTP header.
    /// `requestBody(grant:)` excludes credentials; caller attaches the header
    /// returned by `basicAuthHeader()`.
    case basic
}
```

### `Sources/OAuth2Client/Base64.swift` (NEW, ~80 LOC)

Inline base64 encoder supporting both standard (with `=` padding, `+/`) and url (no padding, `-_`) variants. ~50 LOC core; ~30 LOC for `urlEncode` wrapper.

```swift
// SPDX-License-Identifier: Apache-2.0 WITH LLVM-exception
// Copyright (c) 2026 The bare-swift Project Authors.

@usableFromInline
internal enum Base64 {
    private static let standardAlphabet: [UInt8] = Array(
        "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/".utf8
    )
    private static let urlAlphabet: [UInt8] = Array(
        "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-_".utf8
    )

    /// Standard base64 with `=` padding, `+/`.
    @inlinable
    internal static func encode(_ bytes: [UInt8]) -> String {
        encode(bytes, alphabet: standardAlphabet, padding: true)
    }

    /// URL-safe base64 with no padding, `-_`.
    @inlinable
    internal static func urlEncode(_ bytes: [UInt8]) -> String {
        encode(bytes, alphabet: urlAlphabet, padding: false)
    }

    @inlinable
    internal static func encode(_ bytes: [UInt8], alphabet: [UInt8], padding: Bool) -> String {
        if bytes.isEmpty { return "" }
        var out: [UInt8] = []
        let groupCount = bytes.count / 3
        let remainder = bytes.count % 3
        out.reserveCapacity((groupCount + (remainder > 0 ? 1 : 0)) * 4)

        for g in 0..<groupCount {
            let i = g * 3
            let b0 = UInt32(bytes[i])
            let b1 = UInt32(bytes[i + 1])
            let b2 = UInt32(bytes[i + 2])
            let triplet = (b0 << 16) | (b1 << 8) | b2
            out.append(alphabet[Int((triplet >> 18) & 0x3F)])
            out.append(alphabet[Int((triplet >> 12) & 0x3F)])
            out.append(alphabet[Int((triplet >> 6) & 0x3F)])
            out.append(alphabet[Int(triplet & 0x3F)])
        }
        if remainder == 1 {
            let b0 = UInt32(bytes[groupCount * 3])
            let triplet = b0 << 16
            out.append(alphabet[Int((triplet >> 18) & 0x3F)])
            out.append(alphabet[Int((triplet >> 12) & 0x3F)])
            if padding { out.append(0x3D); out.append(0x3D) }  // ==
        } else if remainder == 2 {
            let b0 = UInt32(bytes[groupCount * 3])
            let b1 = UInt32(bytes[groupCount * 3 + 1])
            let triplet = (b0 << 16) | (b1 << 8)
            out.append(alphabet[Int((triplet >> 18) & 0x3F)])
            out.append(alphabet[Int((triplet >> 12) & 0x3F)])
            out.append(alphabet[Int((triplet >> 6) & 0x3F)])
            if padding { out.append(0x3D) }  // =
        }
        return String(decoding: out, as: UTF8.self)
    }
}
```

### `Sources/OAuth2Client/PKCE.swift` (NEW)

```swift
// SPDX-License-Identifier: Apache-2.0 WITH LLVM-exception
// Copyright (c) 2026 The bare-swift Project Authors.

import Crypto
import Foundation  // for Data bridging to swift-crypto

/// RFC 7636 § 4.3 code_challenge_method.
public enum PKCEMethod: Sendable, Equatable {
    /// `code_challenge = code_verifier` (RFC 7636 § 4.2). For legacy clients.
    case plain
    /// `code_challenge = BASE64URL-ENCODE(SHA256(ASCII(code_verifier)))` (RFC 7636 § 4.2).
    case s256

    /// The RFC 7636 `code_challenge_method` parameter value.
    public var rfc7636Name: String {
        switch self {
        case .plain: return "plain"
        case .s256:  return "S256"
        }
    }
}

extension OAuth2Client {
    /// PKCE (RFC 7636) primitives for the authorization-code grant.
    public enum PKCE: Sendable {
        /// Generate a random PKCE `code_verifier` per RFC 7636 § 4.1.
        ///
        /// `byteCount` is the number of random bytes; the verifier string is
        /// the base64url encoding (no padding). Default 32 bytes → 43-character
        /// verifier. RFC 7636 § 4.1 mandates `43 ≤ verifier.count ≤ 128`.
        public static func generateVerifier(byteCount: Int = 32) -> String {
            let bytes = OAuth2Client.randomBytes(count: byteCount)
            return Base64.urlEncode(bytes)
        }

        /// Compute the PKCE `code_challenge` per RFC 7636 § 4.2.
        ///
        /// - `.plain`: returns `verifier` verbatim.
        /// - `.s256`: returns `base64url(SHA256(verifier.utf8))`.
        public static func challenge(for verifier: String, method: PKCEMethod) -> String {
            switch method {
            case .plain:
                return verifier
            case .s256:
                let digest = SHA256.hash(data: Data(verifier.utf8))
                return Base64.urlEncode(Array(digest))
            }
        }
    }

    /// Generate a random URL-safe token suitable for OAuth `state` or OIDC
    /// `nonce` parameters. Default 16 bytes → 22-character base64url string.
    public static func randomToken(byteCount: Int = 16) -> String {
        Base64.urlEncode(randomBytes(count: byteCount))
    }

    /// Generate `count` cryptographically secure random bytes via Swift's
    /// `SystemRandomNumberGenerator` (CSPRNG on macOS/Linux).
    @usableFromInline
    internal static func randomBytes(count: Int) -> [UInt8] {
        var bytes = [UInt8](repeating: 0, count: count)
        var rng = SystemRandomNumberGenerator()
        for i in 0..<count {
            bytes[i] = UInt8.random(in: 0...255, using: &rng)
        }
        return bytes
    }
}
```

### `Sources/OAuth2Client/OAuth2Client.swift` (extend)

Replace v0.1 struct + init + requestBody with extended v0.2 version. Add `authorizationURL` + `basicAuthHeader` methods.

Key changes:
1. Add `public let clientAuthMethod: ClientAuthMethod` field.
2. Add `clientAuthMethod: ClientAuthMethod = .body` init param.
3. Update `requestBody(grant:)`: when `.basic`, skip appending `client_id` + `client_secret`.
4. Add `basicAuthHeader() -> (name: String, value: String)?`:
   - returns nil when `clientAuthMethod == .body`.
   - returns nil when `clientSecret == nil` (no creds to encode).
   - returns `("Authorization", "Basic <base64(formEncode(client_id) ":" formEncode(client_secret))>")` per RFC 6749 § 2.3.1.
5. Add `authorizationURL(authorizationEndpoint:redirectURI:scope:state:nonce:codeChallenge:codeChallengeMethod:responseType:additionalParams:) -> String`. Composes:
   - `authorizationEndpoint`
   - `?` separator (always present since `client_id` + `response_type` + `redirect_uri` are mandatory per RFC 6749 § 4.1.1)
   - `response_type=<v>` (default "code")
   - `client_id=<v>`
   - `redirect_uri=<v>`
   - Optional: scope, state, nonce, code_challenge, code_challenge_method
   - Additional params appended in order

Query params are form-urlencoded via the existing `FormEncoder.encode(_:)` helper.

### `Package.swift` (modify)

Add `swift-crypto` dep:

```swift
.package(url: "https://github.com/apple/swift-crypto.git", from: "3.0.0")
```

Add `.product(name: "Crypto", package: "swift-crypto")` to the `OAuth2Client` target.

## Stream-format / output examples

**AuthorizationURL build:**

```swift
let url = client.authorizationURL(
    authorizationEndpoint: "https://issuer.example/oauth/authorize",
    redirectURI: "https://app.example/cb",
    scope: "openid profile",
    state: "xyz",
    codeChallenge: "E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM",
    codeChallengeMethod: .s256
)
// → "https://issuer.example/oauth/authorize?response_type=code&client_id=myapp&redirect_uri=https%3A%2F%2Fapp.example%2Fcb&scope=openid+profile&state=xyz&code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM&code_challenge_method=S256"
```

**Basic auth header (RFC 6749 § 2.3.1):**

Client: `clientID = "myapp"`, `clientSecret = "s3cret!"`, `clientAuthMethod = .basic`.
- formEncode("myapp") = "myapp" (unreserved chars)
- formEncode("s3cret!") = "s3cret%21" (! → %21)
- Pair: `"myapp:s3cret%21"` (16 bytes)
- base64 (standard, padded): `"bXlhcHA6czNjcmV0JTIx"` (20 bytes — actually let me recompute: bytes are `6D 79 61 70 70 3A 73 33 63 72 65 74 25 32 31` = 15 bytes; 15/3 = 5 groups, no padding; result is 20 chars).
- Header value: `"Basic bXlhcHA6czNjcmV0JTIx"`

(Brainstorm phase confirmed exact-byte test vector at execution.)

## Test plan (~25 tests)

### `Tests/OAuth2ClientTests/Base64Tests.swift` (NEW, ~5 tests)

1. `Base64.encode([]) == ""` (empty)
2. `Base64.encode([0x4D, 0x61, 0x6E]) == "TWFu"` ("Man" — no padding needed, 3 bytes → 4 chars)
3. `Base64.encode([0x4D]) == "TQ=="` (1 byte → 2 chars + `==`)
4. `Base64.encode([0x4D, 0x61]) == "TWE="` (2 bytes → 3 chars + `=`)
5. `Base64.urlEncode(bytes-containing-+-and-/-in-standard-output) == bytes-with-_-and-_-substituted-and-no-=`

### `Tests/OAuth2ClientTests/PKCETests.swift` (NEW, ~6 tests)

1. `PKCE.generateVerifier()` produces a 43-character URL-safe string (32 bytes → 43 chars after base64url no-padding).
2. `PKCE.generateVerifier(byteCount: 48)` produces a 64-character string.
3. `PKCE.challenge(for: v, method: .plain) == v` (identity).
4. **RFC 7636 § 4.2 worked example**: `PKCE.challenge(for: "dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk", method: .s256) == "E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM"`.
5. `PKCEMethod.s256.rfc7636Name == "S256"`.
6. `PKCEMethod.plain.rfc7636Name == "plain"`.

### `Tests/OAuth2ClientTests/AuthorizationURLTests.swift` (NEW, ~10 tests)

1. Minimal URL: only authorization endpoint + redirectURI → `?response_type=code&client_id=...&redirect_uri=...`.
2. URL with scope.
3. URL with state.
4. URL with nonce.
5. URL with code_challenge + code_challenge_method.
6. URL with all optional params.
7. URL with additionalParams (custom).
8. Custom responseType (e.g., "code id_token").
9. Special characters in redirectURI are form-urlencoded (`:` → `%3A`, `/` → `%2F`).
10. Spaces in scope are form-urlencoded (space → `+`).

### `Tests/OAuth2ClientTests/ClientAuthMethodTests.swift` (NEW, ~5 tests)

1. `ClientAuthMethod.body` (default) — requestBody includes `client_id` and `client_secret` (v0.1 byte-equal regression).
2. `ClientAuthMethod.basic` — requestBody excludes `client_id` and `client_secret`.
3. `ClientAuthMethod.basic` + clientSecret present → `basicAuthHeader()` returns `("Authorization", "Basic ...")` with correct RFC 6749 § 2.3.1 encoding.
4. `ClientAuthMethod.body` → `basicAuthHeader()` returns nil.
5. `ClientAuthMethod.basic` + clientSecret nil → `basicAuthHeader()` returns nil (no creds to encode).

### Extension to `OAuth2ClientTests.swift` (~3 tests)

1. v0.1 default init still works (no clientAuthMethod arg) — byte-equal regression.
2. `OAuth2Client.randomToken()` produces a 22-character URL-safe string (16 bytes → 22 chars).
3. `OAuth2Client.randomToken(byteCount: 32)` produces a 43-character string.

Total: ~25-29 new tests. Combined with v0.1's 26 tests = ~51-55 tests total.

## Acceptance criteria

- v0.2.0 ships with all four v0.2 features.
- v0.1 APIs unchanged. `OAuth2Client(tokenEndpoint:clientID:clientSecret:)` continues to work (default `clientAuthMethod = .body`).
- `requestBody(grant:)` byte-equality preserved for `.body` auth method (regression).
- `parseResponse(_:)` byte-for-byte unchanged.
- swift-crypto dep added.
- RFC 7636 § 4.2 worked-example test passes (S256 challenge calculation correct).
- CI green on macOS + Linux, first try.
- DocC includes `### Authorization flow (v0.2+)` topic group.
- CHANGELOG + README updated.

## Migration

**Additive only — non-breaking.** v0.1 callers' code compiles and behaves identically.

## Decisions locked

- New `ClientAuthMethod` enum + `clientAuthMethod` init param (default `.body` preserves v0.1).
- `basicAuthHeader()` returns tuple `(name: String, value: String)` — composable with any HTTP transport.
- RFC 6749 § 2.3.1: URL-encode client_id + client_secret BEFORE base64 in Basic auth.
- `authorizationURL` returns String. No URL type in bare-swift.
- Default `responseType = "code"`. Allow custom string for forward-compat (e.g., "code id_token" for hybrid flow).
- PKCE namespace lives at `OAuth2Client.PKCE.{generateVerifier, challenge}`.
- `PKCEMethod` is a top-level public enum.
- `randomToken` is a static method on `OAuth2Client` (not in PKCE namespace — useful for state/nonce, not just PKCE).
- Random bytes via Swift stdlib's `SystemRandomNumberGenerator` (CSPRNG on macOS + Linux). No swift-crypto dep for randomness.
- swift-crypto dep added for SHA256 (PKCE S256 challenge).
- Base64 inlined (~80 LOC, supports standard + url variants). No swift-base64 dep.
- No new `OAuth2ClientError` cases — invalid PKCE method / empty client_id etc. are caller bugs, not runtime errors.
- Foundation-in-internal-only pattern (PKCE.swift imports Foundation for Data bridging to swift-crypto). Public API stays Foundation-free.

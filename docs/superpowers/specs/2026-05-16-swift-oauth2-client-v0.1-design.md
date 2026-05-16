# swift-oauth2-client v0.1 Design

**Phase 20 Tranche 20A** (RFC-0025). New package — 49th in the ecosystem.

**Date:** 2026-05-16

---

## Goal

Add an OAuth 2.0 token-endpoint client primitive: emit `application/x-www-form-urlencoded` request bodies for three grant types (`client_credentials`, `authorization_code` with PKCE, `refresh_token`), parse RFC 6749 § 5.1 success and § 5.2 error JSON responses, return a typed `TokenResponse` or throw typed errors. Caller-driven HTTP transport (matches the trinity's pattern). Closes the 8-rejection deferral on OAuth 2.0 and adds the composition layer that ties swift-jwt-verify + swift-bearer + swift-basic-auth into a complete auth-client stack.

## Scope (Shape A — single tranche, v0.1 covers token request + parse)

**In scope:**
- `OAuth2Client` struct: `(tokenEndpoint, clientID, clientSecret?)` config; `requestBody(grant:) -> Bytes` + `parseResponse(_:) throws(OAuth2ClientError) -> TokenResponse`.
- `GrantType` enum: `.clientCredentials(scope:)`, `.authorizationCode(code:redirectURI:codeVerifier:)`, `.refreshToken(_:scope:)`.
- `TokenResponse` struct: standard RFC 6749 § 5.1 fields.
- `OAuth2ClientError` typed-throws enum: `.invalidResponse`, `.oauthError(code:description:)`, `.malformedJSON`.
- Internal `FormEncoder` (`application/x-www-form-urlencoded`).
- Internal `JSONScanner` (minimal top-level field extractor).
- 26 tests across 6 suites.

**Out of scope (deferred to v0.2+):**
- Authorization URL building (caller does it).
- PKCE verifier/challenge generation (caller does it via swift-crypto).
- State / nonce generation (caller does it).
- Token caching / lifecycle / auto-refresh (caller manages).
- OIDC-specific helpers (compose with swift-jwt-verify directly).
- Implicit / password / device-code grants.
- mTLS / `private_key_jwt` client authentication.
- HTTP transport (caller wires).

**Anti-goals:**
- No Foundation in `Sources/OAuth2Client` or test target.
- No swift-base64 dep (no base64 needed in v0.1).
- No swift-uri dep (hand-roll form encoding).
- No swift-json dep (hand-roll JSON scanning).
- No swift-crypto dep (PKCE is caller's job).
- Single direct dep: swift-bytes.

---

## Module structure

```
Sources/OAuth2Client/
  Documentation.docc/
    OAuth2Client.md
  OAuth2Client.swift            ← top-level doc enum + main struct (~80 LOC)
  GrantType.swift               ← enum + form-field emission helper (~80 LOC)
  TokenResponse.swift           ← struct (~25 LOC)
  OAuth2ClientError.swift       ← typed-throws enum (~30 LOC)
  FormEncoder.swift             ← internal x-www-form-urlencoded encoder (~50 LOC)
  JSONScanner.swift             ← internal minimal JSON object field extractor (~100 LOC)
Tests/OAuth2ClientTests/
  OAuth2ClientTests.swift       ← 3 tests
  GrantTypeTests.swift          ← 5 tests
  RequestBodyTests.swift        ← 5 tests
  ParseResponseTests.swift      ← 6 tests
  FormEncoderTests.swift        ← 3 tests
  JSONScannerTests.swift        ← 4 tests
```

Total source: ~365 LOC. Tests: ~500 LOC across 6 suites, 26 tests.

---

## Public API

```swift
public struct OAuth2Client: Sendable {
    public let tokenEndpoint: String      // for documentation/discoverability
    public let clientID: String
    public let clientSecret: String?      // nil for public clients

    public init(tokenEndpoint: String, clientID: String, clientSecret: String? = nil)

    /// Emit application/x-www-form-urlencoded body bytes for the configured grant.
    /// Caller wires the HTTP POST (Content-Type: application/x-www-form-urlencoded).
    public func requestBody(grant: GrantType) -> Bytes

    /// Parse HTTP response body bytes. Detects both RFC 6749 § 5.1 success
    /// and § 5.2 error shapes. Throws on OAuth error or malformed bytes.
    public func parseResponse(_ body: Bytes) throws(OAuth2ClientError) -> TokenResponse
}

public enum GrantType: Sendable {
    case clientCredentials(scope: String? = nil)
    case authorizationCode(code: String, redirectURI: String, codeVerifier: String? = nil)
    case refreshToken(_ token: String, scope: String? = nil)
}

public struct TokenResponse: Sendable, Equatable {
    public let accessToken: String
    public let tokenType: String
    public let expiresIn: Int?
    public let refreshToken: String?
    public let scope: String?
}

public enum OAuth2ClientError: Error, Equatable, Sendable {
    /// Response was not a JSON object, or required success fields missing.
    case invalidResponse
    /// OAuth-spec error response per RFC 6749 § 5.2.
    case oauthError(code: String, description: String?)
    /// JSON parse failure (malformed bytes / not a valid top-level JSON object).
    case malformedJSON
}
```

### Request body field order

Deterministic for testability. Per grant:

**`.clientCredentials(scope:)`** →
1. `grant_type=client_credentials`
2. `client_id=<enc>`
3. `client_secret=<enc>` (only if `clientSecret != nil`)
4. `scope=<enc>` (only if non-nil)

**`.authorizationCode(code:, redirectURI:, codeVerifier:)`** →
1. `grant_type=authorization_code`
2. `code=<enc>`
3. `redirect_uri=<enc>`
4. `code_verifier=<enc>` (only if non-nil)
5. `client_id=<enc>`
6. `client_secret=<enc>` (only if `clientSecret != nil`)

**`.refreshToken(_:, scope:)`** →
1. `grant_type=refresh_token`
2. `refresh_token=<enc>`
3. `scope=<enc>` (only if non-nil)
4. `client_id=<enc>`
5. `client_secret=<enc>` (only if `clientSecret != nil`)

### Client authentication

v0.1 emits `client_id` + `client_secret` in the request body per RFC 6749 § 2.3.1 (which allows body parameters; widely supported). Adopters who need HTTP Basic auth instead:
1. Construct `OAuth2Client(..., clientSecret: nil)` so the secret isn't emitted in the body.
2. Build the `Authorization: Basic base64(client_id:client_secret)` header using swift-basic-auth + swift-base64.

v0.2+ may add a `ClientAuthMethod` knob to switch between `.body` and `.basic`. Out of scope for v0.1.

### `parseResponse` flow

1. Probe bytes for top-level `{` (after whitespace). If not JSON-object-shaped → throw `.malformedJSON`.
2. Look for `"error"` key. If present:
   - Extract `"error"` value (required string) and `"error_description"` (optional string).
   - Throw `.oauthError(code: ..., description: ...)`.
3. Look for `"access_token"` key. If missing → throw `.invalidResponse`.
4. Extract `access_token` (required), `token_type` (required), `expires_in` (optional int), `refresh_token` (optional string), `scope` (optional string).
5. If `token_type` missing → throw `.invalidResponse`.
6. Return `TokenResponse(...)`.

Order matters: error-shape check before success-shape check (some servers return 200 with error body — rare but possible).

---

## Internal types

### `FormEncoder`

Hand-rolled `application/x-www-form-urlencoded` per HTML5 § 4.10.21.7 / WHATWG URL § 5.2:

```swift
internal enum FormEncoder {
    /// Encode a string as a form-urlencoded value.
    /// Unreserved characters (A-Z a-z 0-9 - . _ *) pass through.
    /// Space → "+". Everything else → "%HH" (uppercase hex).
    static func encode(_ s: String) -> [UInt8] {
        var out: [UInt8] = []
        out.reserveCapacity(s.utf8.count)
        for byte in s.utf8 {
            switch byte {
            case 0x30...0x39,           // 0-9
                 0x41...0x5A,           // A-Z
                 0x61...0x7A,           // a-z
                 0x2A, 0x2D, 0x2E, 0x5F:  // * - . _
                out.append(byte)
            case 0x20:                   // space
                out.append(0x2B)         // +
            default:
                out.append(0x25)         // %
                out.append(hexDigit(byte >> 4))
                out.append(hexDigit(byte & 0x0F))
            }
        }
        return out
    }

    private static func hexDigit(_ n: UInt8) -> UInt8 {
        n < 10 ? 0x30 + n : 0x41 + n - 10
    }

    /// Append a `key=value` pair to the buffer, prefixing with `&` if non-empty.
    static func appendField(_ buffer: inout Bytes, key: String, value: String) {
        if !buffer.isEmpty { buffer.append(0x26) }  // &
        buffer.append(contentsOf: Array(key.utf8))
        buffer.append(0x3D)                          // =
        buffer.append(contentsOf: encode(value))
    }
}
```

~50 LOC including comments.

### `JSONScanner`

Minimal JSON object scanner — mirrors v0.1 swift-jwt-verify's `extractAlg` pattern. Finds top-level string OR integer values for known keys without a full JSON parse.

```swift
internal enum JSONScanner {
    /// Find the string value for `key` at the top level of a JSON object.
    /// Returns nil if the key is missing or the value isn't a string.
    /// Throws .malformedJSON on completely-malformed input.
    static func string(forKey key: String, in bytes: Bytes) throws(OAuth2ClientError) -> String? {
        // 1. Find `"<key>"` in the bytes.
        // 2. Skip whitespace, expect `:`.
        // 3. Skip whitespace, expect `"`.
        // 4. Read until unescaped `"`.
        // 5. Unescape \", \\, \/, \n, \r, \t, \b, \f.
        // 6. Return decoded string.
    }

    /// Find the integer value for `key`. Returns nil if missing or not a number.
    static func int(forKey key: String, in bytes: Bytes) throws(OAuth2ClientError) -> Int? {
        // Similar to string(); reads digits (and optional minus sign).
    }

    /// Probe for top-level `{` (after whitespace). Throws .malformedJSON if not found.
    static func validateJSONObject(_ bytes: Bytes) throws(OAuth2ClientError) {
        // ...
    }
}
```

**Handles:** top-level `{...}` shape, `"key":"value"` pairs, `"key":N` for integers, basic backslash escapes (`\"`, `\\`, `\/`, `\n`, `\r`, `\t`, `\b`, `\f`), Unicode whitespace skipping.

**Does NOT handle:** nested objects, arrays as values, `\u####` escapes, scientific notation. v0.1 token responses don't need these. Future v0.2 with non-standard server fields can graduate to swift-json.

~100 LOC.

---

## Test plan (26 tests, 6 suites)

### OAuth2ClientTests (3)
1. Init stores `tokenEndpoint`, `clientID`, `clientSecret`.
2. Init with nil `clientSecret` is allowed (public client) and roundtrips.
3. `requestBody` is deterministic across calls.

### GrantTypeTests (5)
1. `.clientCredentials(scope: nil)` — no scope field in emitted body.
2. `.clientCredentials(scope: "read write")` — scope field present and encoded.
3. `.authorizationCode(..., codeVerifier: "abc")` — `code_verifier` field present.
4. `.authorizationCode(..., codeVerifier: nil)` — `code_verifier` field absent.
5. `.refreshToken("rt-abc", scope: "openid")` — scope present.

### RequestBodyTests (5)
1. `.clientCredentials` confidential client produces exact bytes:
   `grant_type=client_credentials&client_id=my-app&client_secret=hunter2`
2. `.clientCredentials` with scope: appends `&scope=read+write` (space → `+`).
3. `.authorizationCode` with all fields produces exact bytes including `code_verifier`.
4. `.authorizationCode` public client (no secret) omits `client_secret` field.
5. `.refreshToken` produces exact body with `grant_type=refresh_token&refresh_token=...&client_id=...`.

### ParseResponseTests (6)
1. Minimal success: `{"access_token":"abc","token_type":"Bearer"}` → `TokenResponse(accessToken: "abc", tokenType: "Bearer", expiresIn: nil, refreshToken: nil, scope: nil)`.
2. Full success: `{"access_token":"abc","token_type":"Bearer","expires_in":3600,"refresh_token":"rt","scope":"openid email"}` → all fields populated.
3. Error response: `{"error":"invalid_grant","error_description":"bad code"}` → throws `.oauthError("invalid_grant", "bad code")`.
4. Error without description: `{"error":"invalid_client"}` → throws `.oauthError("invalid_client", nil)`.
5. Malformed bytes (`"not json"`) → throws `.malformedJSON`.
6. Missing required `access_token` (`{"token_type":"Bearer"}`) → throws `.invalidResponse`.

### FormEncoderTests (3)
1. Unreserved characters pass through: `FormEncoder.encode("abc-._*123") == [97,98,99,45,46,95,42,49,50,51]`.
2. Space encodes as `+`: `FormEncoder.encode("hello world") == [104,101,108,108,111,43,119,111,114,108,100]`.
3. Special characters encode as `%HH`: `FormEncoder.encode("a/b&c=d") == ...` (each non-unreserved encoded).

### JSONScannerTests (4)
1. Finds top-level string: `string(forKey:"name", in: {"name":"alice"}) == "alice"`.
2. Finds top-level integer: `int(forKey:"age", in: {"age":42}) == 42`.
3. Returns nil for missing key: `string(forKey:"missing", in: {"a":"b"}) == nil`.
4. Handles backslash-escaped quote: `string(forKey:"q", in: {"q":"a\\"b"}) == "a\"b"`.

All tests Foundation-free. Vectors as Swift literals.

---

## CHANGELOG entry

```markdown
## [0.1.0] - 2026-05-16

### Added
- `OAuth2Client` struct — Sendable + value-type OAuth 2.0 token-endpoint client. `(tokenEndpoint, clientID, clientSecret?)` config; `requestBody(grant:)` emits `application/x-www-form-urlencoded` bytes; `parseResponse(_:)` parses RFC 6749 § 5.1 success or § 5.2 error JSON responses.
- `GrantType` enum: `.clientCredentials(scope:)`, `.authorizationCode(code:redirectURI:codeVerifier:)`, `.refreshToken(_:scope:)`. PKCE supported via `codeVerifier:` parameter.
- `TokenResponse` struct: `accessToken`, `tokenType`, `expiresIn?`, `refreshToken?`, `scope?`.
- `OAuth2ClientError` typed-throws enum: `.invalidResponse`, `.oauthError(code:description:)`, `.malformedJSON`.
- 26 tests across 6 suites covering struct shape, per-grant body emission, exact-bytes request bodies, success/error response parsing, FormEncoder spec cases, JSONScanner field extraction.

### Dependencies
- swift-bytes 0.1.0 — single direct dep.
- **Foundation-free.** No swift-crypto, no swift-base64, no swift-uri, no swift-json. All form encoding and JSON scanning hand-rolled (~150 LOC across FormEncoder + JSONScanner).

### Client authentication
- v0.1 emits `client_id` + `client_secret` in the request body per RFC 6749 § 2.3.1.
- Adopters needing HTTP Basic auth: construct with `clientSecret: nil` and build the `Authorization: Basic` header externally (swift-basic-auth + swift-base64 are in the ecosystem).
- `ClientAuthMethod` knob deferred to v0.2.

### Out of scope for v0.1 (deferred to v0.2+)
- Authorization URL building (caller composes via swift-uri or string templates).
- PKCE verifier/challenge generation (caller computes via swift-crypto + swift-base64).
- State / nonce generation.
- Token caching / lifecycle / auto-refresh.
- OIDC-specific helpers (ID-token verification composes with swift-jwt-verify).
- Implicit / password / device-code grants.
- mTLS / `private_key_jwt` client authentication.
- HTTP transport (caller wires).

### Phase 20
- Tranche 20A of [RFC-0025](https://github.com/bare-swift/bare-swift/blob/main/rfcs/0025-phase-20-anchor-swift-oauth2-client.md). Closes the 8-rejection deferral on OAuth 2.0; adds the composition layer that ties swift-jwt-verify + swift-bearer + swift-basic-auth into a complete auth-client stack.
```

---

## README outline

- Tagline: "RFC 6749 OAuth 2.0 token-endpoint client — Sendable, Foundation-free; caller-driven HTTP transport."
- Install block (`from: "0.1.0"`).
- Usage example: PKCE authorization_code flow (most common real-world usage).
- Usage example: client_credentials flow.
- Usage example: refresh_token flow.
- Scope (in / out / anti-goals).
- Pointer to the auth-tier composition: swift-basic-auth + swift-bearer + swift-jwt-verify + swift-oauth2-client.

---

## DocC catalog

`Sources/OAuth2Client/Documentation.docc/OAuth2Client.md`:

- Overview: 2-paragraph description matching the goal section.
- Quickstart code block.
- Topics:
  - **Client** — `OAuth2Client`
  - **Grants** — `GrantType`
  - **Response** — `TokenResponse`
  - **Errors** — `OAuth2ClientError`
- No cross-package symbol links (per [[feedback-docc-cross-package]]).

---

## CI considerations

Standard reusable workflow with `docc-target: OAuth2Client`. Sanitizers ON. Strict-concurrency mandatory. **Pre-emptive Pages setup via the codified three-API-call sequence** (3rd application after Phases 16, 17, 18 set the convention).

---

## Migration

New package — no migration. Adopters writing OAuth 2.0 clients can drop their hand-built token-request boilerplate.

---

## Ship sequencing

1. `bare-swift new`-style scaffold (mkdir + Package.swift + LICENSE + NOTICE + README + CHANGELOG + MAINTAINERS + SECURITY + .gitignore + 3 workflows).
2. `gh repo create bare-swift/swift-oauth2-client`.
3. **Pre-emptive Pages setup** (three-API-call sequence from Gate 16).
4. Implement source files (6 files, ~365 LOC).
5. Implement test suites (6 suites, 26 tests).
6. CHANGELOG + README + DocC catalog.
7. Push, watch CI, fix if needed.
8. Tag v0.1.0; watch Release + Publish docs.
9. Umbrella: add swift-oauth2-client to `bare-swift/packages/index.json` (49th package, tier `http`).
10. Memory closure (`project_phase20_20a_shipped.md`).

Total Phase 20 calendar: ~1-1.5 days wall-clock (RFC-0025's estimate).

---

## Decisions log (locked)

- **Shape A** (single tranche, v0.1 covers token request + parse).
- **Hand-rolled FormEncoder + JSONScanner** (no swift-uri / swift-json deps).
- **Single dep: swift-bytes**. Foundation-free entirely.
- **Client auth in body parameters** (v0.1); `.basic` knob deferred to v0.2.
- **GrantType associated values**: `.clientCredentials(scope:)`, `.authorizationCode(code:redirectURI:codeVerifier:)`, `.refreshToken(_:scope:)`. Scope on clientCredentials + refreshToken; not on authorizationCode (scope is established at authorization-URL time).
- **Module name `OAuth2Client`** (matches package name minus `swift-` prefix).
- **Foundation-free**: includes generation script if any (no script needed in v0.1).
- **Tier**: `http` (matches swift-jwt-verify, swift-bearer, swift-basic-auth entries in packages/index.json).
- **TokenResponse.scope is the raw server string** (caller splits on space if needed; matches RFC 6749 § 3.3 wire format).
- **error response detection precedes success detection** in parseResponse (some servers return 200 + error body).
- **No new umbrella tier.**

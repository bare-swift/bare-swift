# RFC-0025 — Phase 20 anchor: swift-oauth2-client v0.1

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-16 |
| Resolution | Accepted 2026-05-16 — Phase 20 plans may begin authoring. Single new package: swift-oauth2-client. |

## Summary

Anchor Phase 20 on **swift-oauth2-client v0.1** — a new package providing RFC 6749 OAuth 2.0 token-endpoint client primitives. Honest-scope v0.1 covers the three most-common grant types (`client_credentials`, `authorization_code` with PKCE, `refresh_token`) via a single `requestToken(grant:)` operation. Closes the 8-rejection deferral on OAuth 2.0 and adds the composition layer that ties swift-jwt-verify + swift-bearer + swift-basic-auth into a complete auth-client stack.

Single-tranche, new package (49th in ecosystem). Caller-driven HTTP transport (matches the trinity's pattern). ~300-500 LOC honest scope.

## Problem

[Gate 19 retrospective](../docs/gates/2026-05-16-gate-19-retrospective.md) item 5 requires "Phase 20 anchor decision recorded as an RFC" before Phase 20 plans begin. The retrospective surveyed nine candidate waves; this RFC formalizes the choice.

**Concrete gap:** the auth tier is now feature-complete on its primitive packages:
- `swift-basic-auth` (Phase 11A) — HTTP Basic credential parsing.
- `swift-bearer` (Phase 11B) — RFC 6750 Bearer tokens + WWW-Authenticate header.
- `swift-jwt-verify` (Phase 11C v0.1 + Phase 19A v0.2) — JWS-compact sign + verify (HS256 + ES256).

**Missing:** the composition layer. Adopters writing OAuth 2.0 clients (the most common real-world use of the auth tier) currently:
- Build their own token-endpoint POST request payload (`grant_type=...&client_id=...&...`).
- Parse the JSON response (`{"access_token":"...","token_type":"...","expires_in":N,"refresh_token":"..."}`).
- Handle OAuth-spec error responses (`{"error":"invalid_grant","error_description":"..."}`).
- Wire all of this through their HTTP client of choice.

Every adopter writes the same ~200 LOC of boilerplate. swift-oauth2-client provides the typed primitive once.

**8-rejection threshold reached on OAuth 2.0.** It has been deferred at every gate since Phase 11 closed:

| Gate | Defer reason |
|---|---|
| 11 | "no concrete adopter demand" |
| 12 | "rejected as Phase 13; no concrete demand" |
| 13 | "rejected as Phase 14; defer until concrete demand" |
| 14 | "rejected as Phase 15; defer until concrete demand" |
| 15 | "still no concrete demand" |
| 16 | "still no concrete demand" |
| 17 | "no concrete demand" |
| 18 | "still no concrete demand" |
| 19 | **9th rejection threshold** — Gate 19 retro recognized this as procedural drift |

Phase 19 broke through the equivalent 7-rejection deferral on JWT signing by recognizing that "wait a week" had become self-perpetuating. Same logic applies to OAuth 2.0: "no concrete adopter demand" has been used 8 times; but **everything in bare-swift is built proactively**, and the auth tier's primitive completion at Phase 19 makes OAuth the natural next composition layer.

## Proposal

### Anchor: swift-oauth2-client (single new package)

**swift-oauth2-client v0.1** — RFC 6749 OAuth 2.0 token-endpoint client. Module name: `OAuth2Client`. Repo: `bare-swift/swift-oauth2-client`.

| API | Description |
|---|---|
| `OAuth2Client` | Sendable struct holding token endpoint URL + client credentials (`client_id` + `client_secret` for confidential clients; bare client_id + PKCE for public clients). |
| `GrantType` | Enum with three cases: `.clientCredentials`, `.authorizationCode(code:codeVerifier:)`, `.refreshToken(_:)`. |
| `OAuth2Client.requestBody(grant:)` | Returns `Bytes` containing `application/x-www-form-urlencoded` request body for the configured grant. Caller wires the HTTP POST. |
| `OAuth2Client.parseResponse(_:)` | Parses `Bytes` (HTTP response body) → `TokenResponse` struct or throws `OAuth2ClientError`. |
| `TokenResponse` | Struct: `accessToken: String`, `tokenType: String`, `expiresIn: Int?`, `refreshToken: String?`, `scope: String?`. |
| `OAuth2ClientError` | Typed-throws enum: `.networkError(reason:)`, `.invalidResponse`, `.oauthError(code:description:)` (e.g., `invalid_grant`), `.unsupportedGrantType`. |

**Caller-driven HTTP transport** matches the trinity's pattern: the package emits `Bytes` for the request body and accepts `Bytes` for the response body. Caller wires the HTTP client (URLSession / async-http-client / NIO / etc.).

### Tranches

Single-anchor phase shape, two reasonable decompositions:

**Shape A (recommended): single tranche, v0.1 covers token request + parse**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 20A | swift-oauth2-client v0.1 | `OAuth2Client` + 3 grant types + `requestBody` / `parseResponse` + `TokenResponse` + `OAuth2ClientError` + 15-20 tests | ~1-1.5 days |

**Shape B (alternate): two tranches if HTTP transport abstraction expands**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 20A | swift-oauth2-client v0.1 | Same as Shape A but `requestBody` is the only output (parseResponse deferred) | ~0.8 day |
| 20B | swift-oauth2-client v0.2 | Response parsing + TokenResponse + OAuth error mapping | ~0.5 day |

Decision deferred to brainstorm phase. Per the now-canonical brainstorm-reality-check pattern, expect Shape B if the JSON parsing surface expands (token response has optional fields; OAuth error responses have a different shape; some servers return non-spec fields).

### Tranche 20A scope sketch

`swift-oauth2-client` v0.1 covers:

- **`OAuth2Client` struct** holding `tokenEndpoint: String` + `clientID: String` + `clientSecret: String?` (nil for public clients using PKCE).
- **`GrantType` enum** with associated values:
  - `.clientCredentials` — no extra parameters.
  - `.authorizationCode(code: String, redirectURI: String, codeVerifier: String?)` — `codeVerifier` is the PKCE verifier (32-byte random base64url-encoded; caller generates it).
  - `.refreshToken(_ token: String)` — refreshes an access token.
- **`requestBody(grant: GrantType) -> Bytes`** — emits `application/x-www-form-urlencoded` body bytes ready for HTTP POST to the token endpoint.
- **`parseResponse(_ body: Bytes) throws(OAuth2ClientError) -> TokenResponse`** — parses the response body as JSON, extracts known fields, returns `TokenResponse`. On OAuth-spec error response (`{"error":"...","error_description":"..."}`), throws `.oauthError(code:description:)`.
- **`TokenResponse` struct** — `accessToken: String` + `tokenType: String` (typically `"Bearer"`) + `expiresIn: Int?` + `refreshToken: String?` + `scope: String?`.
- **`OAuth2ClientError`** typed-throws enum: `.invalidResponse`, `.oauthError(code: String, description: String?)`, `.malformedJSON`, `.unsupportedGrantType`.
- **JSON parsing**: hand-rolled (~50 LOC) or via swift-json (Phase 6+; in-house). Brainstorm phase decides; lean toward swift-json for forward-compat.

**Reuses existing types:**
- `Bytes` from swift-bytes for byte buffers.
- `Base64URL.encode` from swift-jwt-verify... wait, that's internal. v0.1 will need its own PKCE-compatible base64url; or depend on swift-base64 directly. Brainstorm decides.

**Out of scope for v0.1:**
- **Authorization URL building.** Caller builds the URL per RFC 6749 § 4.1.1 (it's a simple `?response_type=code&client_id=...&redirect_uri=...&scope=...&state=...&code_challenge=...&code_challenge_method=S256` template).
- **PKCE challenge generation.** Caller generates the verifier (random) and challenge (`SHA256(verifier)` base64url-encoded) using swift-crypto.
- **State + nonce generation.** Caller does it with swift-crypto's `SystemRandomNumberGenerator`.
- **Token storage / caching.** Caller manages lifecycle.
- **Automatic refresh.** Caller decides when to refresh.
- **OIDC-specific fields** (`id_token`, `nonce`, `c_hash`). Caller handles OIDC via swift-jwt-verify directly.
- **Implicit grant, password grant, device code grant.** v0.2+ if adopter demand surfaces.
- **mTLS client authentication.** v0.2+.
- **`private_key_jwt` client authentication.** Could compose with swift-jwt-verify v0.2 signing in v0.2+.
- **HTTP transport.** Caller wires.

### Why this anchor

1. **Closes the longest-standing deferral in the auth tier.** 8 prior rejections under "no concrete adopter demand"; Gate 19 recognized this as procedural drift equivalent to the JWT-signing deferral broken in Phase 19.
2. **Composes the just-completed auth primitives.** Authorization-code flow uses Bearer tokens (swift-bearer) for the protected-resource request; the access token can be a JWT (verify via swift-jwt-verify); the OAuth issuer signs ID tokens (sign via swift-jwt-verify v0.2). swift-oauth2-client ties them together via the token endpoint.
3. **Net-new package keeps SemVer churn manageable.** Three consecutive existing-package minor bumps (Phases 17, 18, 19). Phase 20 net-new gives adopters' SemVer surface a rest.
4. **Audience continuity.** Same auth/security adopters from Phases 11 + 19.
5. **Honest-scope v0.1.** Token request + parse only; everything else (authorization URL, PKCE generation, state/nonce, token storage) stays the caller's responsibility. ~300-500 LOC matches Phase 16 swift-log-bridge's shape (the most-recent net-new package of similar scope).
6. **Sets up Phase 21+ options.**
   - **swift-jwt-verify v0.3 (ES384 + ES512 + EdDSA OR RS256 if swift-crypto stabilizes RSA)** becomes more pressing as OAuth adopters need additional algorithm support.
   - **swift-oauth2-client v0.2** can add helpers for authorization URL building + PKCE generation if adopter demand surfaces.
   - **swift-distributed-tracing-bridge v0.3 (B3 + Jaeger)** stays available.
   - **swift-idna v0.3 / swift-brotli v0.3 streaming** stay available.
   - **Package rename swift-jwt-verify → swift-jwt** becomes more pressing.

### Out of scope for Phase 20

- **swift-jwt-verify v0.3 (RS256 OR ES384/ES512/EdDSA).** Phase 21+ candidate.
- **swift-distributed-tracing-bridge v0.3 (B3 + Jaeger).** Phase 21+ candidate.
- **swift-idna v0.3 / swift-brotli v0.3 streaming.** Phase 21+ candidates.
- **Package rename `swift-jwt-verify` → `swift-jwt`.** Phase 22+ candidate.
- **OIDC-specific support.** Adopters compose swift-oauth2-client + swift-jwt-verify directly.
- **Authorization URL building / PKCE helpers / state generation.** v0.2+ if adopter demand surfaces.

### Tooling expectations

Phase 20 reuses all Phase 7–19 tooling without extension:

- `bare-swift new` scaffold pattern (matching swift-log-bridge's directory layout).
- **Pre-emptive Pages setup** per Gate 16's codified three-API-call sequence (Phase 16 + Phase 17 successfully validated; Phase 20 will be the 3rd application).
- Strict-concurrency in CI stays mandatory.
- `docc-target: OAuth2Client` opt-in from scaffold time.
- Sanitizers ON (no large static data).
- Cross-library validation at ship time: round-trip request body construction against the RFC 6749 § 4.1.3 / § 4.4.2 / § 6 examples.

### Versioning

| Package | New | Future |
|---|---|---|
| swift-oauth2-client | v0.1 (3 grant types + request body + response parse) | v0.2 = authorization URL builder + PKCE generation helpers + state/nonce helpers if adopter demand surfaces; v0.3 = implicit/password/device-code grants; v0.4 = mTLS + private_key_jwt client auth |

Phase 20 introduces **1 new package**, taking the ecosystem from **48 to 49 packages**.

### Anti-goals

- **No HTTP client integration.** Caller wires `Bytes` to their HTTP client.
- **No URL building.** Caller builds authorization URLs.
- **No PKCE generation.** Caller generates verifier + challenge with swift-crypto.
- **No token caching / lifecycle.** Caller manages.
- **No retroactive changes** to swift-jwt-verify / swift-bearer / swift-basic-auth.
- **No OIDC-specific helpers.** OIDC composes via swift-jwt-verify directly.
- **No new umbrella tier.** Fits existing `auth` (or `http`) tier — brainstorm decides.

## Alternatives considered

### Anchor Phase 20 on swift-jwt-verify v0.3 (RS256)

**Pros:** closes the most-requested algorithm gap; RSA dominates JWT in OIDC.

**Cons:** swift-crypto's RSA API is still in the `_RSA` experimental SPI. Exposing it to adopters before stabilization risks v0.3 → v0.4 breaking churn when swift-crypto promotes the API. Defer to Phase 22+ when swift-crypto stabilizes.

**Verdict:** rejected as Phase 20.

### Anchor Phase 20 on swift-jwt-verify v0.3 (ES384 + ES512 + EdDSA)

**Pros:** clean algorithm expansion; same Signer/Verifier pattern as v0.2.

**Cons:** third consecutive v0.x bump on same package would be aggressive. Phase 21+ candidate after OAuth 2.0 ships.

**Verdict:** rejected as Phase 20.

### Anchor Phase 20 on swift-distributed-tracing-bridge v0.3 (B3 + Jaeger)

**Pros:** rounds out propagation format coverage.

**Cons:** no concrete adopter demand. Third v0.x bump on same package in three phases would be aggressive. Phase 21+ candidate.

**Verdict:** rejected as Phase 20.

### Anchor Phase 20 on swift-idna v0.3 (Bidi + ContextJ + ContextO)

**Pros:** closes Phase 15's documented deferral.

**Cons:** rarely-hit rules; no concrete demand. Phase 21+ candidate.

**Verdict:** rejected as Phase 20.

### Anchor Phase 20 on swift-brotli v0.3 (streaming encoder)

**Pros:** closes Phase 12 v0.2 one-shot-only deferral; 7 prior deferrals (similar count to OAuth 2.0).

**Cons:** narrower audience than OAuth 2.0's auth-tier closure. Phase 21+ candidate.

**Verdict:** rejected as Phase 20.

### Skip ahead, do crypto-adjacent (swift-blake3 / swift-siphash / swift-hkdf)

**Pros:** would complement OAuth 2.0 if adopters need additional crypto primitives.

**Cons:** swift-crypto remains entrenched. **16th consecutive rejection.**

**Verdict:** rejected.

### Anchor Phase 20 on swift-publicsuffix v0.2 (ICANN/PRIVATE split)

**Pros:** closes a quiet Phase 13 deferral.

**Cons:** narrow scope. Better as a v0.1.1 patch.

**Verdict:** rejected as Phase 20 anchor.

### Bundle OAuth 2.0 client with header customization in swift-jwt-verify

**Pros:** could ship OAuth + `kid` header support together.

**Cons:** they're in different packages and have different audiences; bundling creates artificial coupling. Keep them separate; Phase 21+ can ship `kid` in swift-jwt-verify v0.3.

**Verdict:** rejected; single-anchor-phase shape preserved.

## Resolution

**Accepted 2026-05-16.** Phase 20 plans may begin authoring. Single new package: swift-oauth2-client.

Phase 20 closes the 8-rejection deferral on OAuth 2.0 client primitives — the longest-standing deferral in the auth tier alongside the just-broken-through JWT signing deferral. After Phase 20, the bare-swift auth-client story is feature-complete: basic-auth + bearer + jwt-verify (sign + verify) + oauth2-client all compose into the typical OIDC relying-party stack with zero adopter-side boilerplate beyond HTTP transport wiring.

swift-jwt-verify v0.3 (RS256 OR ES384/ES512/EdDSA), swift-distributed-tracing-bridge v0.3 (B3 + Jaeger), swift-idna v0.3, swift-brotli v0.3 streaming, and a possible `swift-jwt` package rename remain on the queue for Phase 21+.

# RFC-0031 — Phase 26 anchor: swift-oauth2-client v0.2 (auth flow start)

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-16 |
| Resolution | Accepted 2026-05-16 — Phase 26 plans may begin authoring. Single existing package: swift-oauth2-client v0.2. |

## Summary

Anchor Phase 26 on **swift-oauth2-client v0.2** — additive non-breaking minor version bump adding the **auth flow start** features deferred from Phase 20 v0.1: authorization URL builder, PKCE verifier/challenge generation, state/nonce generators, and a `ClientAuthMethod` knob. Together v0.1 + v0.2 cover the complete OAuth 2.0 client surface — back half (token endpoint, Phase 20) + front half (auth flow start, Phase 26). Adds swift-crypto dep for SHA256 (PKCE challenge) + random (state/nonce).

Single-tranche; estimated calendar **determined by brainstorm reality-check** per Gate 22 canonical pattern.

## Problem

[Gate 25 retrospective](../docs/gates/2026-05-16-gate-25-retrospective.md) item 5 requires "Phase 26 anchor decision recorded as an RFC" before Phase 26 plans begin. The retrospective surveyed nine candidate waves; this RFC formalizes the choice.

**Concrete gap:** swift-oauth2-client v0.1 (Phase 20A, shipped 2026-05-16) covers the OAuth 2.0 **token endpoint** — `requestBody(grant:)` emits the form-encoded request body, `parseResponse(_:)` parses success/error JSON. The v0.1 CHANGELOG explicitly defers to v0.2+:

- Authorization URL building.
- PKCE verifier/challenge generation.
- State / nonce generation.
- `ClientAuthMethod` knob (HTTP Basic vs body for client credentials).

Adopters wanting the **auth flow start** (front-channel redirect to authorization endpoint) currently compose v0.1 + swift-crypto + swift-base64 externally. v0.2 makes this ergonomic.

**Why now (Phase 26):**
1. v0.1 has ~3 days settle time — well-stabilized. No incidents.
2. Auth-tier coverage incomplete without auth-flow-start features. v0.1 handles only the token-exchange phase.
3. Strong adoption demand: every OIDC adopter needs auth URL building + PKCE. v0.1's "compose externally" workaround is fine for sophisticated callers but ergonomic for none.
4. Strong audience continuity from Phases 11/19/20.

**Why not the alternatives:**
- **Codec-tier v0.4 drain API sweep** — would unblock multi-coding streaming in content-encoding v0.6, but multi-coding HTTP is ~1% of real traffic. 4-package coordinated sweep is heavier; Phase 27+ when multi-coding demand surfaces.
- **Streaming decoders sweep** — symmetric to encode streaming; adopter demand unclear. Phase 27+.
- **swift-brotli v0.4 + swift-deflate v0.4 window carry** — could pair with drain sweep; premature in isolation.
- **B3+Jaeger, idna v0.3, RS-family JWT, package rename** — low demand or blocked.

## Proposal

### Anchor: swift-oauth2-client v0.2 (single existing package, additive minor bump)

**swift-oauth2-client v0.2** — extend the public API with four documented v0.2 deferrals, all non-breaking additive. Existing `OAuth2Client` struct + `GrantType` + `TokenResponse` + `OAuth2ClientError` preserved verbatim.

| Addition | Description |
|---|---|
| `OAuth2Client.authorizationURL(authorizationEndpoint:redirectURI:scope:state:nonce:codeChallenge:codeChallengeMethod:additionalParams:) -> String` | Builds the OAuth 2.0 authorization URL (front-channel redirect target). Composes scheme/host/path + query parameters per RFC 6749 § 4.1.1 (auth code grant). Returns a String (no URL type in bare-swift). |
| `OAuth2Client.PKCE` namespace enum | Public namespace for PKCE primitives. |
| `OAuth2Client.PKCE.generateVerifier(byteCount:Int) -> String` | Generates a random `code_verifier` (RFC 7636 § 4.1 — 43-128 base64url chars). Default 32 bytes → 43-char verifier. Uses swift-crypto's `SystemRandomNumberGenerator` or equivalent secure random. |
| `OAuth2Client.PKCE.challenge(for verifier: String, method: PKCEMethod) -> String` | Computes `code_challenge` per RFC 7636 § 4.2. `PKCEMethod` enum: `.plain` / `.s256`. S256 uses swift-crypto's SHA256. |
| `OAuth2Client.PKCEMethod` enum | `.plain`, `.s256`. Maps to RFC 7636 § 4.3 `code_challenge_method` strings. |
| `OAuth2Client.randomToken(byteCount:Int) -> String` | Random URL-safe token generator. Used for state + nonce. Default 16 bytes → 22-char base64url string. |
| `ClientAuthMethod` enum | `.body` (default; v0.1 behavior — client_id/secret in request body per RFC 6749 § 2.3.1) or `.basic` (HTTP Basic header). Threaded through `OAuth2Client` init as an additional param (default `.body` preserves v0.1). |
| Likely new helpers on `OAuth2Client` | `OAuth2Client.basicAuthHeader() -> (name: String, value: String)?` if `clientAuthMethod == .basic` — returns the `Authorization: Basic <base64>` header tuple for callers to attach. (Exact API TBD by brainstorm.) |

### Key API shape (subject to brainstorm refinement)

```swift
import OAuth2Client

// 1. Auth flow start: build authorization URL + state + PKCE.
let state = OAuth2Client.randomToken()
let nonce = OAuth2Client.randomToken()
let verifier = OAuth2Client.PKCE.generateVerifier()
let challenge = OAuth2Client.PKCE.challenge(for: verifier, method: .s256)

let authURL = OAuth2Client.authorizationURL(
    authorizationEndpoint: "https://issuer.example/oauth/authorize",
    redirectURI: "https://app.example/callback",
    scope: "openid profile email",
    state: state,
    nonce: nonce,
    codeChallenge: challenge,
    codeChallengeMethod: .s256,
    additionalParams: [:]
)
// → "https://issuer.example/oauth/authorize?response_type=code&client_id=...&..."

// 2. (After redirect callback) Token exchange (v0.1 path, unchanged).
let client = OAuth2Client(
    tokenEndpoint: "https://issuer.example/oauth/token",
    clientID: "myapp",
    clientSecret: "secret",
    clientAuthMethod: .basic       // NEW v0.2 param; default .body preserves v0.1
)
let body: Bytes = client.requestBody(grant: .authorizationCode(
    code: codeFromRedirect,
    redirectURI: "https://app.example/callback",
    codeVerifier: verifier
))
// caller posts body to token endpoint; if clientAuthMethod == .basic,
// caller also attaches client.basicAuthHeader() to the request
let token = try client.parseResponse(responseBody)
```

### Decomposition

**Shape A (recommended pending reality-check): single tranche, all four v0.2 features.**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 26A | swift-oauth2-client v0.2 | AuthorizationURL builder + PKCE primitives + randomToken + ClientAuthMethod + ~25-30 tests | ~1-2 hours (pending reality-check) |

**Shape B (alternate): two tranches if scope/dep concerns split cleanly.**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 26A | swift-oauth2-client v0.2 | AuthorizationURL builder + ClientAuthMethod (no crypto dep) | ~30-60 min |
| 26B | swift-oauth2-client v0.3 | PKCE + randomToken (adds swift-crypto dep) | ~30-60 min |

Decision deferred to brainstorm. Shape A is preferred (single dep bump, single tranche, full v0.2 surface). Shape B is a fallback if a strong reason to delay the swift-crypto dep surfaces in brainstorm.

### Test surface

| Test category | Scope |
|---|---|
| `authorizationURL` round-trip | Build URL with various param combinations; parse query back; verify each field present + encoded correctly — 5-7 tests |
| Query-string encoding (`%`-encoding of redirect URIs, scopes with spaces, additional params) | 2-3 tests |
| `PKCE.generateVerifier` produces 43-character URL-safe strings | 1 test (statistical bounds; deterministic byte-count → exact-length output) |
| `PKCE.challenge(plain)` returns verifier verbatim (RFC 7636 § 4.2) | 1 test |
| `PKCE.challenge(.s256)` matches expected SHA256 base64url for a fixed verifier | 1 test (RFC 7636 § 4.2 worked example) |
| `randomToken` produces non-empty URL-safe strings of expected byte-count → char-count | 1-2 tests |
| `ClientAuthMethod.body` (default) — v0.1 behavior preserved byte-for-byte (regression) | 1-2 tests |
| `ClientAuthMethod.basic` — `basicAuthHeader()` returns correct `Basic <base64(client_id:client_secret)>` value | 2-3 tests |
| `.basic` excludes client_id/secret from request body | 1-2 tests |
| `.basic` with `clientSecret: nil` — returns nil or throws? — TBD by brainstorm | 1 test |
| Integration: full flow end-to-end (build URL → exchange code → parse response, using mocked transport) | 1 test |

Estimated 18-25 new tests. Total package tests post-26A: 26 v0.1 + 18-25 = 44-51.

### Acceptance criteria

- v0.2.0 ships with all four v0.2 features.
- v0.1 APIs unchanged. `OAuth2Client(tokenEndpoint:clientID:clientSecret:)` continues to work (default `ClientAuthMethod = .body`).
- `requestBody(grant:)` byte-equality preserved for `.body` auth method (regression-tested).
- `parseResponse(_:)` byte-for-byte unchanged.
- swift-crypto dep added (first time for swift-oauth2-client).
- CI green on macOS + Linux, first try.
- DocC includes `### Authorization flow (v0.2+)` topic group.
- CHANGELOG + README updated with auth URL + PKCE examples.

### Out of scope (deferred to v0.3+)

- **Token caching / lifecycle / auto-refresh.** Adopters compose with their own storage.
- **OIDC-specific helpers** (ID-token claim validation — composes with swift-jwt-verify).
- **Implicit / password / device-code grants.** Implicit is deprecated; password discouraged; device-code is rare.
- **mTLS / `private_key_jwt` client authentication.**
- **HTTP transport.** Caller continues to wire.
- **JWT-based client authentication** (`client_assertion`).

### Migration (v0.1 → v0.2)

**Additive only — non-breaking.** All v0.1 APIs unchanged:
- `OAuth2Client(tokenEndpoint:clientID:clientSecret:)` continues to work (the new `clientAuthMethod` parameter has a default value of `.body`).
- `requestBody(grant:)` byte-equality preserved for `.body` auth method.
- `parseResponse(_:)` unchanged.
- `OAuth2ClientError` may add 1-2 new cases (additive).

Adopters opt into v0.2 features by calling the new `authorizationURL`, `PKCE`, `randomToken`, and `clientAuthMethod: .basic` paths.

### Risk

**Medium. First swift-crypto dep on swift-oauth2-client.**

| Risk | Mitigation |
|---|---|
| swift-crypto dep brings Foundation-internal usage (already a pattern in other bare-swift packages) | Foundation-in-internal-only pattern (Phase 11+); public API stays Foundation-free. |
| `SystemRandomNumberGenerator` vs `SymmetricKey(size:)` for secure random — choose | Brainstorm decides; both are stable. swift-crypto's `SymmetricKey(size:.bits(N))` gives raw bytes directly. |
| PKCE base64url encoding without `=` padding | Implement inline (~30 LOC) OR depend on swift-base64. v0.1 was swift-base64-free (hand-rolled FormEncoder); v0.2 may continue that pattern or add the dep. TBD by brainstorm. |
| `ClientAuthMethod.basic` requires base64-encoding `client_id:client_secret` | Same encoding question as PKCE base64url. Inline vs swift-base64 dep. |
| Test for SHA256 base64url challenge needs a fixed verifier vector | RFC 7636 § 4.2 provides a worked example: verifier `"dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk"` → challenge `"E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM"`. |
| `OAuth2ClientError` may need `.invalidAuthorizationURL` or `.invalidPKCEMethod` cases | Likely not — both can be checked at config time with simple guards or fatalError on impossible inputs. TBD by brainstorm. |

## Alternatives considered

See [Gate 25 retrospective](../docs/gates/2026-05-16-gate-25-retrospective.md) § Item 5 for the full candidate matrix.

## Dependencies

- swift-oauth2-client v0.2 will **add swift-crypto** (first time on this package) for SHA256 + secure random.
- swift-base64 dep is **TBD** — may add for base64url encoding of PKCE challenge + HTTP Basic credentials, or inline a small base64url helper.
- swift-bytes dep unchanged.

## References

- [RFC 6749](https://www.rfc-editor.org/rfc/rfc6749) — OAuth 2.0. § 4.1.1 (auth URL), § 2.3.1 (client authentication).
- [RFC 7636](https://www.rfc-editor.org/rfc/rfc7636) — PKCE. § 4.1 (verifier), § 4.2 (challenge), § 4.3 (method).
- [Gate 25 retrospective](../docs/gates/2026-05-16-gate-25-retrospective.md) — anchor decision rationale.
- [RFC-0001](0001-conventions-and-quality-bars.md) — bare-swift conventions.
- [RFC-0025](0025-phase-20-anchor-swift-oauth2-client.md) — Phase 20 anchor (swift-oauth2-client v0.1) defining the v0.1 surface that v0.2 augments.
- swift-oauth2-client v0.1 CHANGELOG — documents v0.2+ deferrals.

## Decision

**Accepted 2026-05-16.** Phase 26 anchored on **swift-oauth2-client v0.2 (auth flow start)**. Brainstorm + plan + execute via the bare-swift inline-execution pattern. Shape A (single tranche, all four features) is the working assumption; Shape B fallback only if brainstorm uncovers strong reasons to split the dep addition.

Per-tranche calendar estimate **deferred to brainstorm reality-check** per Gate 22's canonical pattern.

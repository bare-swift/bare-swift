# RFC-0034 — Phase 29 anchor: swift-oauth2-client v0.3 (token caching + refresh + OIDC ID-token helpers)

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-17 |
| Resolution | Accepted 2026-05-17 — Phase 29 plans may begin authoring. Single existing package: swift-oauth2-client v0.3. |

## Summary

Anchor Phase 29 on **swift-oauth2-client v0.3** — additive non-breaking minor version bump adding token caching + auto-refresh + OIDC ID-token claim helpers. Closes documented v0.2 deferrals. Completes the OAuth 2.0 / OIDC client story for typical adopter use cases.

Single-tranche; estimated ~1-2 hours (net-new-features calibration). Per-tranche estimate **deferred to brainstorm reality-check** per Gate 22 canonical pattern.

## Problem

[Gate 28 retrospective](../docs/gates/2026-05-17-gate-28-retrospective.md) item 5 requires "Phase 29 anchor decision recorded as an RFC" before Phase 29 plans begin. The retrospective surveyed nine candidate waves; this RFC formalizes the choice.

**Concrete gap:** swift-oauth2-client v0.1 (Phase 20A) shipped the token endpoint; v0.2 (Phase 26A) added auth flow start (auth URL builder + PKCE + state/nonce + ClientAuthMethod). Both v0.1 + v0.2 CHANGELOGs explicitly defer to v0.3+:

- **Token caching / lifecycle / auto-refresh** — every OIDC adopter needs this; v0.1 + v0.2 leave it to manual composition.
- **OIDC ID-token helpers** — ID-token claim validation composes externally with swift-jwt-verify; v0.3 makes the composition ergonomic.

**Why now (Phase 29):**
1. v0.2 has ~1.5 days settle time — well-stabilized.
2. Auth-tier story is feature-complete at the protocol level (basic-auth + bearer + jwt-verify + oauth2-client v0.1+v0.2); v0.3 completes it at the **ergonomic** level (every OIDC adopter today does manual caching/refresh wiring).
3. Modest scope (~1-2 hours). Wrapper-pattern doesn't apply (net-new features) but composition is straightforward.

**Why not the alternatives (full survey in Gate 28 retro § Item 5):**
- **Streaming decoders sweep** — 12-20hr coordinated sweep; adopter demand unclear. Phase 30+.
- **swift-brotli v0.5 + swift-deflate v0.5 window carry** — ratio improvements; not blocking. Defer.
- **B3+Jaeger, idna v0.3** — no demand.
- **RS-family JWT** — STILL BLOCKED on swift-crypto `_RSA` SPI.
- **Package rename** — premature.

## Proposal

### Anchor: swift-oauth2-client v0.3 (single existing package, additive minor bump)

**swift-oauth2-client v0.3** — extend v0.2's public API with token caching, auto-refresh, and OIDC ID-token helpers. v0.1 + v0.2 surface preserved verbatim.

| Addition | Description |
|---|---|
| `TokenStorage` protocol (Sendable) | Pluggable storage for cached `TokenResponse` per OAuth issuer + scope tuple. Methods: `load() -> CachedToken?`, `store(_:)`, `clear()`. Default in-memory implementation provided. |
| `CachedToken` struct | `TokenResponse` + `obtainedAt: Date` + `expiresAt: Date?` (computed from `expiresIn`). Sendable. |
| `OAuth2Client.cache(storage:)` or wrapper struct | Wraps a base `OAuth2Client` with token storage. Returns cached token if not expired (with optional refresh-before-expiry threshold); otherwise triggers refresh path. |
| `OAuth2Client.refreshGrant(_:)` | Convenience for the refresh-token flow. v0.1 already supports `.refreshToken(_:scope:)` grant via `requestBody(grant:)`; v0.3 layers a high-level refresh method on top. |
| `OIDCTokenResponse` struct (or extension method) | Surfaces ID token bytes + parsed claims. Composition with swift-jwt-verify is documented; v0.3 ships either: (a) a `parseIDToken(_:verifier:)` helper that wraps jwt-verify, OR (b) typed `idToken: String?` field on `TokenResponse` + caller verifies externally. **TBD by brainstorm.** |
| Possible new error case | `OAuth2ClientError.tokenExpired` or `.refreshFailed(...)` — TBD. |

### Key API shape (subject to brainstorm refinement)

```swift
// 1. Token caching with auto-refresh.
let storage = OAuth2Client.InMemoryTokenStorage()
let client = OAuth2Client(
    tokenEndpoint: "https://issuer.example/oauth/token",
    clientID: "myapp",
    clientSecret: "secret"
)
let cached = OAuth2Client.Cached(client: client, storage: storage)

// First call: triggers token exchange + caches.
let token1 = try await cached.token(for: .clientCredentials(scope: "read"))
// Second call within expiry window: returns cached.
let token2 = try await cached.token(for: .clientCredentials(scope: "read"))
// (After expiry): auto-refresh via refresh_token if available; else re-exchange.

// 2. OIDC ID-token claim helpers.
let response: TokenResponse = ... // from token exchange
let idTokenString = response.idToken  // if v0.3 surfaces it
let claims = try OIDCClaims.parse(idTokenString)
// Caller verifies claims.iss / aud / exp / nonce / sub independently.
```

### Decomposition

**Shape A (recommended pending reality-check): single tranche, all three features.**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 29A | swift-oauth2-client v0.3 | TokenStorage + Cached wrapper + OIDCClaims (or idToken accessor) + ~15-20 tests | ~1-2 hours (pending reality-check) |

**Shape B (alternate): two-tranche if scope splits.**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 29A | swift-oauth2-client v0.3 | TokenStorage + Cached wrapper + auto-refresh | ~1 hour |
| 29B | swift-oauth2-client v0.4 | OIDCClaims / ID-token helpers | ~30-60 min |

Shape A is preferred. Shape B fallback if OIDC helpers expand scope.

### Critical brainstorm questions (locked at brainstorm time)

1. **TokenStorage thread-safety:** Sendable protocol so cached state can be shared across tasks? Or single-task only? **Likely:** Sendable; default in-memory uses an actor or Mutex.
2. **Refresh strategy:**
   - Eager refresh on access if token expires within N seconds (threshold)?
   - Lazy refresh on 401 response from resource server?
   - Both?
   - **Likely:** eager-by-threshold (e.g., 60s); document lazy-on-401 as caller's responsibility.
3. **Identity of cached entries:** key by (tokenEndpoint, clientID, scope, grant_type)? Or just per-client? **Likely:** per-grant-and-scope to support multiple concurrent scopes.
4. **ID-token surface:** add `idToken: String?` to `TokenResponse` (parse from JSON `id_token` field if present)? Add `OIDCClaims.parse(_:)` helper? Both? **Likely:** both — `TokenResponse.idToken` is a string field; `OIDCClaims.parse(_:)` extracts payload claims (composes with swift-jwt-verify for verification).
5. **Does Cached wrapper take an HTTP transport (callback)?** v0.1/v0.2 are caller-driven HTTP (no transport in package). For caching+refresh, the wrapper MUST be able to trigger token exchange — which requires HTTP. **Likely:** caller provides a `transport: @Sendable (URLRequest-equivalent) async throws -> ResponseBytes` closure. Or: the Cached wrapper is "passive" — caller is responsible for triggering refresh and calling `cached.store(_:)`; cache only reads + tracks expiry.

### Test surface (~15-20 tests)

**TokenStorage:**
1. In-memory storage stores + loads + clears.
2. Multi-key storage (different grants/scopes) doesn't collide.

**Cached wrapper:**
3. First access triggers exchange + stores.
4. Within expiry: returns cached without re-exchange.
5. Within refresh-threshold: triggers refresh path.
6. Past expiry: triggers refresh path.
7. Refresh failure: throws or falls back to re-exchange (TBD).

**OIDC ID-token:**
8. `TokenResponse.idToken` field parses from JSON.
9. `OIDCClaims.parse(_:)` extracts `iss`, `aud`, `exp`, `nonce`, `sub` from JWT payload (composes with swift-jwt-verify for signature verification).
10. Missing ID-token in TokenResponse returns nil (not throws).

**Regression:**
11-15. v0.1 + v0.2 byte-for-byte preserved for `.body` auth method, `requestBody(grant:)`, `parseResponse(_:)`, `authorizationURL(...)`, PKCE primitives.

### Acceptance criteria

- v0.3.0 ships with all three features.
- v0.1 + v0.2 APIs unchanged.
- swift-crypto dep unchanged from v0.2 (already there).
- swift-jwt-verify dep added IF v0.3 ships built-in JWT verification for ID tokens; else just documentation. **TBD by brainstorm.**
- CI green on macOS + Linux, first try.
- DocC includes `### Token caching (v0.3+)` and `### OIDC (v0.3+)` topic groups.
- CHANGELOG + README updated.

### Out of scope (deferred to v0.4+)

- **mTLS / `private_key_jwt` client authentication.**
- **JWT-based `client_assertion`.**
- **Device-code / implicit / password grants.**
- **Distributed token cache backends** (Redis, Keychain, etc.) — `TokenStorage` is a protocol; caller wires storage.
- **HTTP transport integration.**
- **JWKS endpoint fetching for ID-token verification keys.**

### Migration (v0.2 → v0.3)

**Additive only — non-breaking.** v0.1 + v0.2 APIs unchanged.

### Risk

**Medium. First package addition of token-caching semantics.**

| Risk | Mitigation |
|---|---|
| TokenStorage protocol design surface; once shipped, hard to evolve | Brainstorm reality-checks Sendable + minimum-method protocol shape; default impl simple. |
| Auto-refresh thread-safety with concurrent token requests | If Sendable storage uses actor / Mutex, single source of truth. Caller is single-task-per-cache by default. |
| Date handling without Foundation | swift-oauth2-client public API is Foundation-free in v0.2 (uses string/int for expiresIn). v0.3 may need Foundation-internal-only for Date semantics — same pattern as PKCE.swift in v0.2. |
| ID-token JWT parsing — swift-jwt-verify dep adds weight | TBD by brainstorm: either ship the dep (full-feature) or document the composition (zero-dep). |
| Wrong abstraction for Cached wrapper (too tightly coupled vs too loose) | Brainstorm reality-checks two API shapes: (a) passive cache (read/store/expiry) + caller-orchestrates refresh; (b) active wrapper that triggers refresh internally via caller-provided transport. **Likely (a)** — passive cache aligns with v0.1/v0.2's caller-driven-HTTP pattern. |

## Alternatives considered

See [Gate 28 retrospective](../docs/gates/2026-05-17-gate-28-retrospective.md) § Item 5 for the full candidate matrix.

## Dependencies

- swift-oauth2-client v0.3 dep on swift-bytes unchanged.
- swift-crypto dep unchanged (from v0.2).
- swift-jwt-verify dep **TBD by brainstorm** (could ship for ID-token verification, or document composition).
- Possible Foundation-internal-only for Date semantics (Sendable Date alternative TBD).

## References

- [RFC 6749](https://www.rfc-editor.org/rfc/rfc6749) — OAuth 2.0. § 6 (refresh token grant).
- [RFC 7519](https://www.rfc-editor.org/rfc/rfc7519) — JWT.
- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html) — ID Token specification.
- [Gate 28 retrospective](../docs/gates/2026-05-17-gate-28-retrospective.md) — anchor decision rationale.
- [RFC-0025](0025-phase-20-anchor-swift-oauth2-client.md) — Phase 20 (oauth2-client v0.1).
- [RFC-0031](0031-phase-26-anchor-swift-oauth2-client-v0.2-auth-flow.md) — Phase 26 (oauth2-client v0.2).
- swift-oauth2-client v0.1 + v0.2 CHANGELOGs — document v0.3+ deferrals (token caching, OIDC helpers).

## Decision

**Accepted 2026-05-17.** Phase 29 anchored on **swift-oauth2-client v0.3 (token caching + refresh + OIDC ID-token helpers)**. Brainstorm + plan + execute via the bare-swift inline-execution pattern.

Per-tranche calendar estimate **deferred to brainstorm reality-check** per Gate 22's canonical pattern (~1-2 hours expected; net-new-features bracket).

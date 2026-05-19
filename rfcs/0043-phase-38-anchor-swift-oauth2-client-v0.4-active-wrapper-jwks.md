# RFC-0043 — Phase 38 anchor: swift-oauth2-client v0.4 (active wrapper + JWKS)

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-19 |
| Resolution | Accepted 2026-05-19 — Phase 38 plans may begin authoring. Single existing package: swift-oauth2-client v0.4. |

## Summary

Anchor Phase 38 on **swift-oauth2-client v0.4** — additive non-breaking minor bump adding two auth-tier features:
1. **Active token wrapper** — orchestrates `TokenStorage` cache + automatic refresh + retry logic on top of the v0.1-v0.3 caller-driven-HTTP foundation. Adopters delegate the cache + refresh lifecycle to the library; the library calls back to the adopter's HTTP closure only when a new token is needed.
2. **JWKS support** — `JWKS` struct + key lookup by `kid` + caller-provided HTTP closure for JWKS fetch. Composes with `OIDCClaims` (v0.3) for adopter-driven ID-token signature verification.

Picks up a new narrative arc after the **codec-tier true-memory-streaming arc closure** (Phase 30 → 37). Auth-tier story extends.

Per-tranche calendar **deferred to brainstorm reality-check** per Gate 22 canonical pattern. Likely 1.5-3 hr per net-new-features bucket (1-2 hr from Phase 29 calibration; possibly higher for two features in one tranche).

## Problem

[Gate 37 retrospective](../docs/gates/2026-05-19-gate-37-retrospective.md) item 5 requires "Phase 38 anchor decision recorded as an RFC" before Phase 38 plans begin. The retrospective surveyed eight candidate waves; this RFC formalizes the choice.

**Concrete gap:** swift-oauth2-client v0.3 ships caller-driven-HTTP token caching + OIDC ID-token claim extraction. Adopters wanting **automatic** token lifecycle (cache → check expiry → refresh-if-needed → retry-on-401) must write the orchestration themselves. Adopters wanting **signature-verified** ID tokens must fetch JWKS + look up by `kid` + verify externally; the library provides claims parsing but no JWKS plumbing.

**Why now (Phase 38):**
1. **Codec-tier arc closed (Phase 37).** Phase 38 picks up a new narrative.
2. **v0.3 settle time: 8 phases.** v0.3 shipped Phase 29 (Phase 32-37 were codec-tier). swift-oauth2-client has had reasonable observation time.
3. **Concrete adopter use cases.** Active-wrapper handles the most common OAuth2 client pattern (web app calling a downstream API with auto-refresh). JWKS handles the most common OIDC verification pattern.
4. **Composes with existing v0.3 surface.** Active-wrapper uses v0.3's `TokenStorage` + `CachedToken`; JWKS composes with v0.3's `OIDCClaims`.
5. **6 deferrals on the queue.** Phase 32-37 each deferred swift-oauth2-client v0.4 in favor of codec-tier work; the deferral is now ready to resolve.

**Why not the alternatives (full survey in Gate 37 retro § Item 5):**
- **swift-brotli v0.7 + swift-deflate v0.7 window carry** — ratio improvement; not blocking; no adopter demand.
- **Codec one-shot/streaming unification** — internal cleanup; no adopter-visible value.
- **B3 + Jaeger, idna v0.3** — long-running correct-deferrals; no demand.
- **RS-family JWT** — STILL BLOCKED on swift-crypto `_RSA` SPI.
- **Crypto-adjacent** — 29th rejection.
- **Package rename** — premature.

## Proposal

### Anchor: swift-oauth2-client v0.4 (single existing package, additive minor bump)

**swift-oauth2-client v0.4** — additive features. v0.1-v0.3 public APIs unchanged.

| Addition | Description |
|---|---|
| `OAuth2Client.activeWrapper(...)` (or analogous type) | Orchestrates `TokenStorage` cache + refresh. Caller provides: (a) an HTTP closure for token-endpoint calls, (b) a `TokenStorage` (defaults to `InMemoryTokenStorage`), (c) credentials. Wrapper exposes `currentToken() async throws -> CachedToken` that checks cache → refreshes if expired (within threshold) → stores → returns. Sendable, Foundation-free. |
| `JWKS` struct | Parsed JSON Web Key Set per RFC 7517. Fields: array of JWKs (each with `kty`, `kid`, `use`, `alg`, plus key-type-specific material like `n`/`e` for RSA, `x`/`y`/`crv` for EC). Sendable, Foundation-free JSON parsing via inline JSON scanner. |
| `JWKS.parse(_:)` | Parse JWKS JSON into the struct. Throws on malformed input. |
| `JWKS.key(forKid:)` | Look up a key by `kid`. Returns `JWK?`. |
| `JWKS` fetch via caller-provided HTTP closure | Match v0.1-v0.3 caller-driven-HTTP pattern. Free function or static method that takes a URL + HTTP closure and returns a `JWKS`. |
| Possibly new error case(s) | TBD by brainstorm. Likely `OAuth2ClientError.invalidJWKS` + `OAuth2ClientError.tokenRefreshFailed` (or reuse existing cases). |

### Brainstorm decision: Shape A (combined tranche) vs Shape B (split into 2)

Per 5-instance brainstorm-empowered-by-RFC scope simplification pattern:

- **Shape A (recommended): single tranche, active-wrapper + JWKS.** Both features in one tranche. ~1.5-3 hr.
- **Shape B (alternate): 2 tranches, active-wrapper first then JWKS.** ~1-1.5 hr per tranche.

Brainstorm reality-checks based on:
- Whether active-wrapper needs JWKS for any internal verification (likely not — ID-token signature verification is caller-driven; active-wrapper just refreshes access tokens via the token endpoint, not ID-token JWTs).
- Whether the two features compose cleanly in a single CHANGELOG narrative (likely yes — both are "complete the OAuth2 + OIDC story" extensions).

Shape A is the working assumption; Shape B is the safer split if reality-check uncovers entanglement.

### Test surface

| Test category | Scope | Estimated count |
|---|---|---|
| Active-wrapper happy path | First call refreshes + caches; subsequent calls within expiry use cache | 2-3 tests |
| Active-wrapper refresh logic | Token within refresh threshold triggers refresh | 2 tests |
| Active-wrapper concurrent calls | Multiple concurrent `currentToken()` calls don't trigger multiple refreshes (single-flight) | 1-2 tests |
| JWKS parsing | Valid RSA + EC JWKS; missing `kid`; malformed JSON | 3-4 tests |
| JWKS lookup | Found / not-found by `kid` | 1-2 tests |
| JWKS fetch via caller closure | Closure called with correct URL; returns parsed JWKS | 1-2 tests |
| Composition test | Active-wrapper + JWKS + OIDCClaims pipeline (illustrative, not exhaustive) | 1 test |

Estimated 11-16 new tests. Higher than typical because two features in one tranche.

### Acceptance criteria

- v0.4.0 ships with active-wrapper + JWKS.
- v0.1-v0.3 APIs unchanged.
- All existing tests pass without modification.
- CI green on macOS + Linux, first try.
- DocC includes new topic groups for active-wrapper and JWKS.
- CHANGELOG v0.4.0 entry documents both features + new error cases (if any).
- **DocC discipline:** cross-package symbol refs use **single-backtick** (notably any refs to swift-crypto or swift-jwt-verify).
- Sendable + Foundation-free constraints honored.

### Out of scope (deferred to v0.5+ or later phases)

- **JWT signature verification.** The library extracts claims (v0.3) + provides JWKS lookup (v0.4); adopters verify signatures via swift-jwt-verify (which has its own progress; HS256 + ES256 ship; ES384/ES512/EdDSA shipped in Phase 21; RS-family blocked on swift-crypto `_RSA` SPI).
- **mTLS support** — caller-driven via the HTTP closure; no library-level support needed in v0.4.
- **private_key_jwt client authentication** — Phase 39+ candidate.
- **Device-code grant** — Phase 39+ candidate.
- **Token introspection / revocation endpoints** — Phase 39+ candidate.

### Migration (v0.3 → v0.4)

**Additive only — non-breaking.** All v0.1-v0.3 APIs unchanged.

### Risk

**Medium per brainstorm-locked shape:**
- **Shape A (combined)**: Medium risk. Two net-new features in one tranche. Mitigation: brainstorm reality-check before committing; if scope is too large, split to Shape B.
- **Shape B (split)**: Lower risk. Each feature in its own tranche.

| Risk | Mitigation |
|---|---|
| Active-wrapper single-flight semantics are non-trivial to get right | Use an actor for the wrapper; concurrent `currentToken()` calls serialize through the actor. Test concurrent calls explicitly. |
| JWKS JSON parsing needs Foundation-free implementation | Inline JSON scanner already exists in v0.3 (used for `TokenResponse` parsing). Extend it for JWKS shape. |
| RSA key material requires Foundation-free big-int handling | If we need to surface `n` / `e` as integers, we can defer that to a `JWK.rsaPublicKey() -> (n: Bytes, e: Bytes)` accessor that returns raw bytes; integer conversion is caller's responsibility (or via swift-crypto integration). Avoids pulling Foundation. |
| Cross-package DocC ref slip | Single-backtick discipline mandatory. Apply memory note as checklist (Gate 31 lesson). |
| Scope creep on Shape A | Brainstorm reality-check enforces scope. If both features can't fit in one tranche cleanly, split to Shape B. |

## Alternatives considered

See [Gate 37 retrospective](../docs/gates/2026-05-19-gate-37-retrospective.md) § Item 5 for the full candidate matrix.

## Dependencies

- swift-bytes ≥ 0.1.0 (unchanged).
- swift-crypto ≥ 3.0 (unchanged from v0.2).
- No new external dependencies.

## References

- [RFC 6749](https://www.rfc-editor.org/rfc/rfc6749) — OAuth 2.0 Authorization Framework.
- [RFC 7517](https://www.rfc-editor.org/rfc/rfc7517) — JSON Web Key (JWK) + JSON Web Key Set (JWKS).
- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html) — OIDC ID-token semantics.
- [Gate 37 retrospective](../docs/gates/2026-05-19-gate-37-retrospective.md) — anchor decision rationale.
- [RFC-0035](0035-phase-29-anchor-swift-oauth2-client-v0.3-token-caching.md) — Phase 29 (oauth2-client v0.3 token caching); the foundation being extended here.
- [feedback_docc_cross_package](MEMORY first-line) — single-backtick for cross-package DocC refs.

## Decision

**Accepted 2026-05-19.** Phase 38 anchored on **swift-oauth2-client v0.4 active wrapper + JWKS**. Brainstorm + plan + execute via the bare-swift inline-execution pattern. Per-tranche calendar estimate **deferred to brainstorm reality-check** per Gate 22's canonical pattern. Shape A vs Shape B decided by brainstorm.

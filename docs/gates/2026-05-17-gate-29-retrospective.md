# Gate 29 Retrospective: Phase 29 → Phase 30

**Date:** 2026-05-17
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 29 (the swift-oauth2-client v0.3 token caching + OIDC ID-token claim helpers wave anchored by [RFC-0034](../../rfcs/0034-phase-29-anchor-oauth2-client-v0.3-caching-refresh-oidc.md)) against the Gate 1–28 criteria template and recommends whether Phase 30 should start. Phase 29 was a **single-tranche existing-package minor bump** (29A swift-oauth2-client v0.3). Shipped cleanly.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All anchored Tranche 29A deliverables shipped, CI green, DocC live, index bumped | ✓ PASS |
| 2 | RFC-0034 commitments fulfilled | ✓ PASS *(zero scope reshape; 13th consecutive clean RFC since Phase 17)* |
| 3 | ≥1 substantive RFC accepted *or documented absence of policy gaps* | ✓ PASS (3 new orchestration patterns codified) |
| 4 | RFC-0001 conventions stress-test against Phase 29 deliverables | ✓ DONE BELOW |
| 5 | Phase 30 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0035 below) |

**Overall:** Items 1–4 pass. Item 5 is open. Phase 30 plans should be drafted but not executed until item 5 closes via RFC-0035.

Phase 29 calendar time was ~1 hour wall-clock (within RFC-0034's 1-2 hour net-new-features bracket).

---

## Item 1 — Phase 29 deliverables shipped ✓ PASS

| Tranche | Package | Tag | New tests | DocC | Index | CI |
|---|---|---|---|---|---|---|
| 29A | swift-oauth2-client | **v0.3.0** | 20 (72 total) | ✓ | ✓ (0.2.0→0.3.0) | green (first-try) |

Single tranche shipped. **Auth-tier story FULLY COMPLETE.**

**Phase 29 final tally:**
- 2 new public surface areas: token caching (TokenStorage + CachedToken + InMemoryTokenStorage) + OIDC ID-token claims (OIDCClaims + TokenResponse.idToken).
- 1 new error case (`OAuth2ClientError.invalidIDToken`).
- 1 internal helper extended (`Base64.urlDecode`).
- ~650 source LOC + ~470 test LOC + ~80 docs LOC change.
- 72 tests across 12 suites (52 v0.2 + 20 new).
- ~1 hour wall-clock — within RFC-0034's 1-2 hour estimate.
- **18-consecutive first-try-clean CI streak** (Phases 16-29).
- **13-consecutive zero-scope-reshape RFC streak** (Phases 17-29).
- v0.1 + v0.2 byte-for-byte preserved (regression-tested).
- **No new external deps** — swift-jwt-verify deliberately deferred (composes externally).

**Auth-tier story FULLY COMPLETE:**
- swift-basic-auth (Phase 11A v0.1) — HTTP Basic credential parsing
- swift-bearer (Phase 11B v0.1) — RFC 6750 Bearer + WWW-Authenticate
- swift-jwt-verify (11C v0.1 + 19A v0.2 + 21A v0.3) — 5 algorithms sign+verify+kid
- swift-oauth2-client (20A v0.1 + 26A v0.2 + **29A v0.3**) — token endpoint + auth flow start + token caching + OIDC ID-token claims

Adopters now have ergonomic full-stack OAuth/OIDC. Only manual composition needed: HTTP transport wiring + JWT signature verification (the latter via swift-jwt-verify, already in the ecosystem).

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists **49 packages** — no count change.

---

## Item 2 — RFC-0034 commitments ✓ PASS (zero scope reshape)

[RFC-0034](../../rfcs/0034-phase-29-anchor-oauth2-client-v0.3-caching-refresh-oidc.md) committed to:

| RFC-0034 commitment | Outcome |
|---|---|
| Existing package swift-oauth2-client v0.2 → v0.3 (additive minor bump) | ✓ shipped. |
| TokenStorage protocol (Sendable, async load/store/clear) | ✓ shipped. |
| CachedToken struct with expiry tracking | ✓ shipped (clock-agnostic via UInt64 Unix epoch seconds; no Foundation Date in public API). |
| InMemoryTokenStorage actor (default in-process) | ✓ shipped. |
| OIDC ID-token: TokenResponse.idToken field + OIDCClaims.parse helper | ✓ shipped. |
| Passive cache vs active wrapper decision | ✓ **passive cache** chosen during brainstorm reality-check (matches v0.1/v0.2 caller-driven-HTTP pattern). |
| Refresh strategy decision | ✓ caller-orchestrates externally (passive cache). |
| swift-jwt-verify dep decision | ✓ NOT added (claims extraction only; signature verification composes externally). |
| ID-token API surface decision | ✓ both `TokenResponse.idToken: String?` and `OIDCClaims.parse(_:)` helper. |
| Calendar estimate **deferred to brainstorm reality-check** | ✓ honored. Locked at 1-2 hours during brainstorm; actual ~1 hour. |
| ~15-20 tests (RFC range) | ✓ 20 new tests (upper bound). |
| Non-breaking on v0.1+v0.2 | ✓ confirmed. v0.1/v0.2 byte-for-byte preserved. |
| Out of scope: active wrapper, JWKS, mTLS, private_key_jwt, device-code | ✓ all honored. |

**Zero scope reshape.** **13th consecutive clean-from-RFC phase** (Phases 17-29).

**Estimate-vs-actual:** RFC-0034 said "1-2 hours pending reality-check"; actual ~1 hour. Lower bound. **Net-new-features calibration at 2 instances** (Phase 26 + Phase 29).

---

## Item 3 — Revised criterion ✓ PASS

**Pre-Phase 29 (Gate 28 closeout):** RFC-0034 accepted 2026-05-17.

**During Phase 29 execution:** zero RFCs accepted.

Phase 29 surfaced **three new canonical patterns** + reinforced existing patterns:

- **NEW: Passive-cache pattern (Phase 29 codification).** Cache stores tokens + reports expiry; caller orchestrates HTTP refresh externally. Matches the caller-driven-HTTP pattern established in v0.1/v0.2. **Memory:** when a package needs caching and the package doesn't drive its own HTTP layer, prefer passive cache over active wrapper. Avoids transport abstraction creep.
- **NEW: Clock-agnostic-via-UInt64-Unix-epoch pattern (Phase 29 codification).** Store time as `UInt64` Unix epoch seconds; expose `isExpired(at: UInt64, threshold:)` taking "now" from caller. Avoids Foundation Date in public API. Caller provides clock via `Date().timeIntervalSince1970` or any other source. **Memory:** applies to any future time-based API surface that needs Foundation-free public API.
- **NEW: Actor-backed default + protocol-for-pluggability pattern (Phase 29 codification).** Define a Sendable protocol (caller-implementable for adopter-specific backends) + ship an actor-backed default in-process implementation. Adopters can swap in Keychain/Redis/filesystem-backed alternatives. **Memory:** applies whenever a package needs stateful state that adopters might want to customize (caches, queues, connection pools, key stores).
- **Foundation-internal-only pattern continues.** Phase 29 stayed Foundation-free in public API (UInt64 Unix epoch seconds; no Date). swift-crypto dep from v0.2 unchanged.
- **No-new-dep discipline.** RFC-0034 considered adding swift-jwt-verify for ID-token signature verification but the brainstorm reality-check confirmed claims extraction was the actual scope; signature verification composes externally. Pattern: ship the data extraction layer; defer the signature verification layer to composition.
- **Reality-check-before-RFC-estimate at 8/8 successful applications.**
- **Net-new-features-package calibration at 2 instances.**

None warrant a new cross-package RFC. **Documented absence of policy gaps. Criterion satisfied.**

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

| Convention | swift-oauth2-client v0.3 |
|---|---|
| Module name PascalCase | ✓ OAuth2Client |
| Apache-2.0 + LLVM exception SPDX | ✓ |
| Sendable-clean | ✓ all new types Sendable (protocol is Sendable; actor is Sendable; struct is Sendable + Equatable) |
| Single public error enum | ✓ OAuth2ClientError (4 cases: 3 v0.1 + 1 new `invalidIDToken`) |
| Public APIs Foundation-free | ✓ (UInt64 Unix epoch seconds; no Date) |
| README + DocC updated | ✓ |
| CHANGELOG with v0.3.0 entry | ✓ |
| CI green | ✓ first-try |
| DocC bundle | ✓ |
| Sanitizers ON | ✓ |
| Backwards-compat preservation | ✓ v0.1/v0.2 byte-for-byte; TokenResponse.init default idToken: nil; new error case is additive |

### Deviations and findings

**1. Three new canonical patterns codified in one phase.** Phase 29 ships passive-cache + clock-agnostic-UInt64 + actor-backed-default-with-protocol all together. These are tightly related (token caching needs all three). **Memory:** multi-pattern phases happen naturally when a feature requires coordinated design choices; that's fine.

**2. No-new-dep discipline.** swift-jwt-verify was a candidate for ID-token signature verification. Reality-check confirmed extraction-only scope; verification composes externally. **Memory:** when a package needs a feature that "feels like" it should be an integrated dep, ask: "does this need to be a dep, or can it compose externally?" Composition keeps dep graphs narrow.

**3. Foundation-internal-only pattern preserved.** swift-crypto from v0.2 imports Foundation internally for Data bridging. Phase 29 added nothing new on the Foundation front. Public API remains Foundation-free.

**4. Auth-tier story now FULLY complete at the ergonomic level.** The 5-phase arc (Phase 11A bearer/basic-auth, 11C jwt-verify v0.1, 19A jwt-verify v0.2, 20A oauth2-client v0.1, 21A jwt-verify v0.3, 26A oauth2-client v0.2, 29A oauth2-client v0.3) shipped a complete OAuth/OIDC client surface across 4 packages over ~4 weeks. Future auth work is composition refinements (active wrappers, JWKS, mTLS) — adopter-demand-driven.

### Verdict

RFC-0001 held cleanly under Phase 29 stress. Three new patterns codified; no-new-dep discipline reinforced; auth-tier feature-complete.

---

## Item 5 — Phase 30 anchor decision ✗ NOT YET (recommendation)

Phase 30 anchor candidates surveyed:

| Wave | Packages | Risk | Adoption fit | Deferral count |
|---|---|---|---|---|
| **swift-deflate v0.5 (streaming inflate)** | swift-deflate v0.5 | Medium-high (major refactor of Inflater.swift state machine) | High — opens streaming-decode for the codec tier; bottom of the stack (gzip/zlib wrap deflate). | 1 (Phase 28+) |
| swift-gzip v0.4 + swift-zlib v0.4 streaming inflate (2-tranche) | gzip v0.4, zlib v0.4 | Medium — wraps deflate streaming inflate; downstream of swift-deflate v0.5. | 1 each |
| swift-brotli v0.5 streaming inflate | swift-brotli v0.5 | High (Brotli decoder is complex; metablock state machine) | High — complete streaming-decode-side; symmetric to Phase 22 brotli streaming encoder. | 1 |
| Full streaming decoders sweep (deflate + gzip + zlib + brotli in one phase) | 4 packages, 4 tranches | Very high (3-5hr per tranche; ~12-20hr total) | High but heavy — completes the bidirectional streaming HTTP body story. | 1 each |
| swift-brotli v0.5 + swift-deflate v0.5 window carry across chunks | 2 packages | Medium | Low-medium — ratio improvement only. | 5 each (Phases 25-29) |
| swift-oauth2-client v0.4 (active wrapper / JWKS / mTLS / private_key_jwt / device-code) | swift-oauth2-client v0.4 | Medium | Low — v0.3 just shipped same-day; settle time needed. | n/a (no settled deferral) |
| swift-distributed-tracing-bridge v0.3 (B3 + Jaeger) | bridge v0.3 | Low-medium | Low — 10 deferrals; no concrete demand. | 10 |
| swift-idna v0.3 (Bidi + ContextJ + ContextO) | swift-idna v0.3 | Medium | Low — rarely-hit Unicode rules. | 14 |
| RS-family JWT algorithms | swift-jwt-verify v0.4 | High | Low — **STILL BLOCKED** on swift-crypto `_RSA` SPI. | n/a (blocked) |
| Crypto-adjacent | various | Medium-low | Low — 25x rejected. | 25 (rejected) |
| Package rename `swift-jwt-verify` → `swift-jwt` | rename project | Low | Low-medium | n/a |

### Recommendation: **Anchor on swift-deflate v0.5 (streaming inflate)**

Reasons:

1. **Opens the streaming-decode side** of the codec tier. Phase 22-28 shipped streaming-encode; streaming-decode is the symmetric work.
2. **Bottom of the codec stack.** swift-deflate v0.5 unblocks swift-gzip v0.4 streaming decode + swift-zlib v0.4 streaming decode (downstream wrappers). Phase 31+ can build on it.
3. **Modest single-tranche scope.** ~3-5hr wall-clock for Inflater state-machine refactor. Larger than recent wrapper-pattern phases (30-60min) but bounded — fits in a single sitting.
4. **Phase-31+ visibility.** Once deflate v0.5 ships, gzip + zlib streaming decode become 30-60min wrapper-pattern phases (downstream of deflate). brotli streaming decode is independent (Brotli is a separate algorithm). The streaming-decode story closes in 2-3 more phases.
5. **High adopter value.** Streaming decode covers large response bodies (think: streaming JSON parsing, log-tailing over HTTP, server-sent events). Symmetric to streaming encode demand.
6. **Reality-check needed.** Brainstorm should survey v0.1's `Inflater.swift` state machine — current one-shot inflate reads bytes top-to-bottom. Streaming inflate requires:
   - Buffering input bytes that arrive incrementally.
   - State-machine refactor: read-header → read-block-type → read-block-body → read-block-or-trailer.
   - Yield-and-resume semantics: when input is exhausted mid-block, save state; resume when more input arrives.
   - Output buffering: emit decoded bytes incrementally (similar to streaming encoder's drain()).
   - This is **genuine state-machine work** — not a wrapper-pattern phase. Per Gate 26's calibration: wrapper-pattern = 30-60min; state-machine work = 3-5hr or more.
7. **Non-breaking additive minor bump.** All v0.4 APIs unchanged. v0.5 adds:
   - `Deflate.Streaming.Decoder` struct (mirror canonical Streaming.Encoder shape: init, update(_:), finish() throws -> Bytes).
   - Possibly `Deflate.Streaming.Decoder.bytes` partial-output getter (similar to drain).
   - Likely 1-2 new error cases (`decoderFinished`, possibly `streamingNotYetSupported` for unsupported edge cases — TBD by brainstorm).

### Phase 30 tranche sketch (subject to formal RFC + brainstorm reality-check)

**Shape A (recommended pending reality-check): single tranche, full streaming inflate.**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 30A | swift-deflate v0.5 | `Deflate.Streaming.Decoder` (init/update/finish) + Inflater state-machine refactor + ~12-18 tests | ~3-5 hours (pending reality-check) |

**Shape B (alternate): two tranches if state-machine refactor extends.**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 30A | swift-deflate v0.5 | Streaming inflate for stored blocks + fixed-Huffman blocks (BFINAL=0/1) | ~2-3 hours |
| 30B | swift-deflate v0.6 | Streaming inflate for dynamic-Huffman blocks + edge cases | ~2-3 hours |

Decision deferred to brainstorm phase. Shape A is preferred (single tranche, full coverage). Shape B fallback if the dynamic-Huffman state machine is more complex than expected.

### Streaming-decode sweep roadmap (subsequent phases)

After Phase 30 ships swift-deflate v0.5:
- **Phase 31:** swift-gzip v0.4 + swift-zlib v0.4 streaming decode (2 tranches; wrap deflate v0.5 streaming inflate; ~30-60min per tranche).
- **Phase 32:** swift-brotli v0.5 streaming decode (single tranche; 3-5hr; independent of deflate).
- **Phase 33+:** swift-content-encoding v0.7 streaming-decode wiring through HTTP layer.

Total streaming-decode sweep: ~10-15hr spread across 4 phases. Half the "full sweep" cost in Gate 28's estimate (12-20hr) due to wrapper-pattern downstream work.

### Why other waves were rejected for Phase 30

- **Full streaming decoders sweep (4-tranche)** — too heavy for a single phase. Spread across 3 phases (30, 31, 32) per the roadmap.
- **swift-brotli v0.5 streaming inflate first** — independent of deflate; could be Phase 30 alternative. swift-deflate first is preferred because it unblocks gzip/zlib downstream (multi-package leverage).
- **swift-brotli v0.5 + swift-deflate v0.5 window carry** — ratio improvement; not blocking adopters. Defer.
- **swift-oauth2-client v0.4** — v0.3 just shipped same-day; zero settle time.
- **B3 + Jaeger, idna v0.3** — no demand.
- **RS-family JWT** — blocked.
- **Crypto-adjacent** — 25th rejection.
- **Package rename** — premature; Phase 33+.

**Action:** author the anchor decision as **RFC-0035** (Phase 30 anchor: swift-deflate v0.5 streaming inflate) and accept it before Phase 30 plans start.

---

## Open work to clear Gate 29

1. **Write RFC-0035** (Phase 30 anchor: swift-deflate v0.5 streaming inflate). 1-hour task.
2. **Begin Phase 30 brainstorm + reality-check** after RFC-0035 accepts. Brainstorm should:
   - Survey v0.1's `Inflater.swift` state-machine structure (BitReader → header → meta-block-loop → block-type → body).
   - Settle Streaming.Decoder API shape (mirror Streaming.Encoder: init / update(_:) / finish() throws -> Bytes).
   - Settle yield-and-resume semantics: how state is saved when input bytes are exhausted mid-block.
   - Settle output buffering: cumulative output buffer drained at finish? Drain method like encoder side? Yield bytes incrementally on each update?
   - Settle new error case(s): `decoderFinished` (mirror brotli/gzip pattern). Possibly `inputExhausted` for caller errors.
   - Settle test surface: round-trip via Deflate.encode (v0.2 one-shot) + Deflate.Streaming.Decoder (new) → verify decoded bytes match original.

---

## What this retrospective changes for Phase 30

- The plan-then-execute-inline workflow stays.
- **Wrapper-package calibration at 7 instances; net-new-features calibration at 2 instances.** Phase 30 is **state-machine work** (Inflater refactor) — third calibration bucket. Expect 3-5hr per tranche; **not** wrapper-pattern.
- **3 new patterns codified in Phase 29** (passive cache, clock-agnostic UInt64, actor-backed default + protocol). Available for future stateful-state work.
- **Reality-check-before-RFC-estimate at 8/8.** Phase 30 RFC must defer per-tranche estimate to brainstorm.
- **Auth-tier story FULLY COMPLETE.** Future auth work is composition refinements; adopter-demand-driven.
- **Streaming-decode story OPENING.** Phase 30 is the first phase in the streaming-decode sweep arc.
- **Carried forward:** Phase 29 left no documented deferrals beyond v0.4+ candidates already listed (active wrapper, JWKS, mTLS, etc.).

---

## Decision

Gate 29 is **PASSED on items 1, 2, 3, 4** and **NOT YET PASSED on item 5 (open)**. Phase 30 plans should be drafted but not executed until item 5 closes via RFC-0035.

Phase 29 completed the auth-tier story at the ergonomic level. Three new patterns codified (passive cache, clock-agnostic UInt64, actor-backed default with protocol). 18-consecutive first-try-clean CI streak; 13-consecutive zero-scope-reshape RFC streak.

The next step is **RFC-0035 anchoring Phase 30 on swift-deflate v0.5 (streaming inflate)** — opens the streaming-decode side of the codec tier; bottom of the codec stack; sets up Phase 31+ for wrapper-pattern downstream work on gzip + zlib + content-encoding decode wiring. swift-brotli v0.5 streaming decode, swift-brotli v0.5 + swift-deflate v0.5 window carry, swift-oauth2-client v0.4, B3 + Jaeger, idna v0.3, RS-family JWT (when swift-crypto stabilizes `_RSA`), and package rename remain on the queue for Phase 31+.

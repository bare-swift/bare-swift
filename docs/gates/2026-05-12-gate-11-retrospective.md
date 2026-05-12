# Gate 11 Retrospective: Phase 11 → Phase 12

**Date:** 2026-05-12
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 11 (the auth-primitives wave anchored by [RFC-0016](../../rfcs/0016-phase-11-anchor-auth-primitives.md)) against the Gate 1–10 criteria template and recommends whether Phase 12 should start. Phase 11 was a **four-package wave** (11A basic-auth, 11B bearer, 11C jwt-verify, 11D jwt-claims). All four shipped.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All anchored Tranche 11A/B/C/D deliverables shipped at v0.1+, CI green, DocC live, listed | ✓ PASS |
| 2 | RFC-0016 commitments fulfilled (Basic + Bearer + JWS-compact verify + claim validation) | ✓ PASS |
| 3 | ≥1 substantive RFC accepted *or documented absence of policy gaps* | ✓ PASS (documented absence of policy gaps; see below) |
| 4 | RFC-0001 conventions stress-test against Phase 11 deliverables | ✓ DONE BELOW |
| 5 | Phase 12 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0017 below) |

**Overall:** Items 1–4 pass cleanly. Item 5 is open. Phase 12 plans should be drafted but not executed until item 5 closes via RFC-0017.

The roadmap's stop conditions did not trigger: Phase 11 calendar time was ~1 day for all four tranches. No security incidents. No Apple announcements that subsumed planned packages.

---

## Item 1 — Phase 11 deliverables shipped ✓ PASS

| Tranche | Package | Tag | New tests | DocC | Index | CI |
|---|---|---|---|---|---|---|
| 11A | swift-basic-auth | v0.1.0 | — | ✓ | ✓ | green |
| 11B | swift-bearer | v0.1.0 | — | ✓ | ✓ | green |
| 11C | swift-jwt-verify | v0.1.0 | 20 across 6 suites | ✓ | ✓ | green (after `docc-target` fixup) |
| 11D | swift-jwt-claims | v0.1.0 | 38 across 5 suites | ✓ | ✓ | green (after DocC ref + disambiguation fixup) |

All four tranches shipped. The wave closes the HTTP-server primitive stack: a Phase 11–powered server can now decompress the body (Phase 7+10), parse headers (Phase 4+8), **verify identity (Phase 11)**, apply business logic, compress the response (Phase 9), and send it back.

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists **43 packages**.

**Calendar time:** Phase 11 executed in ~1 day wall-clock. swift-basic-auth and swift-bearer were minutes each. swift-jwt-verify was the densest tranche (~330 LOC source + the first swift-crypto-wrapping package since swift-distributed-tracing-bridge in Phase 3). swift-jwt-claims was ~180 LOC.

---

## Item 2 — RFC-0016 commitments ✓ PASS

[RFC-0016](../../rfcs/0016-phase-11-anchor-auth-primitives.md) committed to a four-package wave:

| RFC-0016 commitment | Outcome |
|---|---|
| Tranche 11A: swift-basic-auth (RFC 7617) — ~80 LOC | ✓ shipped at v0.1.0 |
| Tranche 11B: swift-bearer (RFC 6750 + WWW-Authenticate) — ~80 LOC | ✓ shipped at v0.1.0 |
| Tranche 11C: swift-jwt-verify (HS256 / RS256 / ES256 verify) — ~400 LOC | ✓ shipped at v0.1.0 with HS256 + ES256 (RS256 deferred to v0.2 — see deviation below) |
| Tranche 11D *(stretch)*: swift-jwt-claims (RFC 7519 § 4.1) — ~150 LOC | ✓ shipped at v0.1.0 |
| Foundation-free public APIs across the wave | ✓ honored |
| swift-crypto dep ONLY in swift-jwt-verify | ✓ honored — basic-auth, bearer, jwt-claims pull no crypto deps |
| NOTICE attribution per RFC-0001 | ✓ honored |
| Anti-goals: no new umbrella tier, no swift-crypto in baseline, no retroactive sweep | ✓ honored |

**Deviations beyond RFC-0016's stated commitments:**

- **swift-jwt-verify v0.1 covers HS256 + ES256 only, NOT RS256.** RFC-0016 listed HS256/RS256/ES256. Decided during the 11C implementation plan: RS256 requires RSA key import (PEM / DER) which adds material complexity. Deferred to v0.2. HS256 (symmetric) + ES256 (asymmetric) cover the two algorithm shapes; adopters needing RS256 can wait or fall back to a swift-crypto direct call.
- **swift-jwt-claims `validate(_:)` accepts `[UInt8]` not `[String: Any]`.** RFC-0016 § "Tranche 11D scope sketch" flagged the input-type as TBD ("considers using swift-json's `JSONValue` type as the input — to be decided in the implementation plan"). Resolved in design: `[UInt8]` payload bytes for seamless composition with `JWTVerifier.verify(_:).payloadBytes`. JSON parsed internally via swift-json. Cleaner than the alternatives; matches the Phase 8 byte-stream conventions.
- **First bare-swift package to import Foundation internally.** swift-jwt-verify uses `import Foundation` in `Verifier.swift` only, with explanatory comment, to bridge `[UInt8]` ↔ swift-crypto's `Data` API. Public API stays Foundation-free. Convention from RFC-0001 ("internal use is fine") applied — but this is the first package to exercise it. swift-uuid (Phase 1) had set the same precedent earlier; 11C confirms it.

---

## Item 3 — Revised criterion (≥1 RFC accepted *or* documented absence of policy gaps) ✓ PASS

**Pre-Phase 11 (Gate 10 closeout):** RFC-0016 (Phase 11 anchor) accepted 2026-05-12.

**During Phase 11 execution:** zero RFCs accepted.

The revised criterion (introduced at Gate 9, confirmed at Gate 10) accepts "documented absence of policy gaps" as equivalent. Phase 11 surfaced four gotchas during execution; all are memory-saved transferable lessons rather than cross-package policy gaps:

- **Swift 6.3.1 SILGen ownership crash** on `catch let e as ConcreteError` inside a typed-throws function. Workaround: drop the `as` cast; typed-throws guarantees the type. Documented as `feedback_swift_6_3_typed_throws_catch_as` memory.
- **DocC disambiguation suffix drift** between local Xcode docc and Linux CI docc. The `-9rxiu`-style hash differs across toolchains; the structural `->ReturnType` form is stable. Documented as `feedback_docc_disambiguation_suffixes` memory.
- **DocC cross-package reference syntax** (already known from Gate 9). swift-jwt-claims's initial `JWTVerify.verify(_:)` doc reference needed correction from `\`\`X\`\`` to single-backtick. Restated this as a one-line lesson — convention is unchanged.
- **swift-time `Duration` requires explicit `Time.Duration` qualification** when used as a parameter type. Doesn't auto-resolve from `import Time` because `Duration` is also a Swift stdlib type. Per-package adaptation, not a cross-cutting policy.

None of these warrant a new cross-package RFC. **Documented absence of policy gaps; criterion satisfied.**

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

Per-package review:

| Convention | basic-auth | bearer | jwt-verify | jwt-claims |
|---|---|---|---|---|
| Module name PascalCase, no `Swift` prefix | ✓ BasicAuth | ✓ Bearer | ✓ JWTVerify | ✓ JWTClaims |
| Apache-2.0 + LLVM exception SPDX | ✓ | ✓ | ✓ | ✓ |
| NOTICE crediting upstream | ✓ clean-room RFC 7617 | ✓ clean-room RFC 6750 | ✓ clean-room + swift-crypto attribution | ✓ clean-room RFC 7519 |
| Swift 6.0+ tools version | ✓ | ✓ | ✓ | ✓ |
| macOS 14 platform floor | ✓ | ✓ | ✓ | ✓ |
| Sendable-clean by default | ✓ | ✓ | ✓ | ✓ |
| Strict concurrency in CI | ✓ | ✓ | ✓ | ✓ |
| Single public error enum, typed throws | ✓ BasicAuthError | ✓ BearerError | ✓ JWTVerifyError (6 cases) | ✓ JWTClaimsError (8 cases) |
| Public APIs Foundation-free | ✓ | ✓ | ✓ (internal Foundation in Verifier.swift only) | ✓ |
| Repo skeleton from `bare-swift new` | ✓ | ✓ | ✓ | ✓ |
| README tagline + ≤30-line example | ✓ | ✓ | ✓ | ✓ |
| CHANGELOG with v0.x entry per Keep-a-Changelog | ✓ | ✓ | ✓ | ✓ |
| CI green on macOS + Linux | ✓ | ✓ | ✓ | ✓ |
| DocC bundle | ✓ | ✓ | ✓ | ✓ |
| `docc-target` set from initial commit | ✓ | ✓ | ✗ (fixup commit needed) | ✓ (lesson applied from 11C) |

### Deviations and findings

**1. First public package to import Foundation internally (swift-jwt-verify).** RFC-0001 explicitly allows "internal use is fine"; swift-uuid set the silent precedent. swift-jwt-verify is the first package where the import is *visible* in source review (Verifier.swift bridges `[UInt8]` ↔ swift-crypto `Data`). The convention that emerged: plain `import Foundation` with a comment line citing the bridge rationale, NOT `internal import Foundation` (Swift 6 syntax). Matches swift-uuid.

**2. swift-crypto wrapping pattern.** swift-jwt-verify is the second bare-swift package to wrap swift-crypto (after swift-distributed-tracing-bridge in Phase 3). The pattern that emerged: typed-throws Swift wrapper translates swift-crypto's untyped errors into a package-specific error enum; calls `isValidAuthenticationCode` / `isValidSignature` directly without manual constant-time compare; uses x963 uncompressed form (`0x04 || x || y`) for P-256 public keys and raw `r||s` (NOT DER) for ES256 signatures per RFC 7515 § 3.4.

**3. `docc-target: JWTClaims` from initial commit (11D).** 11C hit the strict-DocC failure because shared `package-ci.yml` runs `--warnings-as-errors` across all targets including swift-crypto's stale DocC catalog. 11D applied the lesson preemptively. This becomes the **canonical pattern**: any package with a dep that has its own DocC catalog must set `docc-target` to its own module in `.github/workflows/ci.yml` from the scaffolder output. Worth folding into `bare-swift new`.

**4. Patch-release-shape mini-tranches.** 11A and 11B were each ~30-minute tranches with comparable test surface. The wave's overall shape — small (11A/B) + medium (11C) + small (11D) — proves a four-tranche wave can absorb both small and medium packages without losing momentum.

### Verdict

RFC-0001 held cleanly under Phase 11 stress (first swift-crypto-wrapping package since Phase 3 + first publicly-visible internal Foundation import + first multi-tranche-with-shared-dep wave).

---

## Item 5 — Phase 12 anchor decision ✗ NOT YET (recommendation)

Phase 12 anchor candidates surveyed:

| Wave | Packages | Risk | Adoption fit |
|---|---|---|---|
| **swift-brotli v0.2 (encoder)** | swift-brotli v0.2 + swift-content-encoding v0.4 | Medium-high (~1500 LOC encoder) | **High** — closes Phase 10's decoder-first staging; named by Gate 9 + 10 as strongest next signal. |
| Internationalization | swift-idna, swift-publicsuffix | Medium-high (UTS #46 + #15 + Punycode; PSL data-heavy) | Medium — closes swift-uri's IDNA deferral. |
| OTLP v0.2 cross-signal | swift-otlp-traces v0.2, swift-otlp-logs v0.2, swift-otlp-json | Medium | Medium — narrower audience than encoder. |
| swift-jwt-verify v0.2 (signing + RS256) | swift-jwt-verify v0.2 | Medium | Medium-low — verify-only is the v0.1 design boundary; signing inflates scope. |
| OAuth 2.0 client primitives | swift-oauth2-client | Medium | Low-medium — sits atop swift-bearer; no concrete adopter demand yet. |
| Crypto-adjacent | swift-blake3, swift-siphash, swift-hkdf | Medium-low | Low — ninth rejection candidate. |

### Recommendation: **Anchor on swift-brotli v0.2 encoder**

Reasons:

1. **Closes Phase 10's decoder-first staging.** Phase 10 anchored on swift-brotli v0.1 decoder explicitly with v0.2 encoder as the symmetric follow-on. Phase 9 established the encoder-after-decoder pattern (deflate / gzip / zlib v0.2 encoders after v0.1 decoders); applying it to brotli closes the last remaining asymmetry in the compression tier.
2. **Cited by two consecutive gate retrospectives.** Gate 9 named brotli as the strongest Phase 10 candidate (single-anchor); Gate 10 named brotli encoder as a strong Phase 11 alternative if auth fell through; Gate 11 (this doc) confirms it as Phase 12's natural anchor.
3. **Audience continuity.** Same HTTP server/client implementers Phases 4 / 7 / 8 / 9 / 10 / 11 serve. Encoder makes brotli usable for response compression, completing the symmetric request/response compression story across all five web codings.
4. **Single-anchor phase shape works.** Phase 10 (single-anchor brotli decoder + small content-encoding follow-on) was the cleanest-shipping phase of the ecosystem to date — one focused deliverable, no tranching pressure. Phase 12's natural shape is the same: swift-brotli v0.2 encoder as 12A, swift-content-encoding v0.4 as a small 12B follow-on adding the `br` encode branch.
5. **Risk is well-bounded.** Brotli encoding is more complex than decoding (literal/match/copy decisions, sliding window management, ring buffer, multiple quality tiers) but the encoder's correctness is checkable against the decoder we already have: round-trip vectors. ~1500 LOC encoder, several days. No new tier on the umbrella.
6. **Sets up Phase 13+ options.** Once brotli encoder ships:
   - **Internationalization** (IDNA / PSL) becomes the natural Phase 13 anchor.
   - **OTLP cross-signal v0.2** stays available.
   - **swift-jwt-verify v0.2** (signing + RS256) stays available.

### Phase 12 tranche sketch (subject to formal RFC)

| Tranche | Packages | Estimated calendar |
|---|---|---|
| 12A | swift-brotli v0.2 (encoder; ~1500 LOC) | ~2-3 days |
| 12B *(stretch)* | swift-content-encoding v0.4 (add `br` encode branch) | ~0.5 day |

Total Phase 12 budget: ~3 days wall-clock at observed pace. Resembles Phase 10 in shape (single anchor + small follow-on).

### Why other waves were rejected

- **Internationalization** — still the right Phase 13+ anchor. Data-heavy + spec-heavy (UTS #46 + #15 + Punycode + PSL ~150 KiB). Deserves its own phase.
- **OTLP v0.2 cross-signal** — narrower audience; deferral isn't aging fast.
- **swift-jwt-verify v0.2 (signing)** — fresh package; let v0.1 settle and gather adopter signal before expanding scope.
- **OAuth 2.0 client primitives** — composition layer atop swift-bearer; better to wait for concrete adopter demand.
- **Crypto-adjacent** — ninth rejection. swift-crypto remains entrenched.

**Action:** author the anchor decision as **RFC-0017** (Phase 12 anchor: swift-brotli v0.2 encoder) and accept it before Phase 12 plans start.

---

## Open work to clear Gate 11

1. **Write RFC-0017** (Phase 12 anchor: swift-brotli v0.2 encoder). 1–2 hour task. References this retrospective + Gate 9/10's standing recommendations.
2. **Begin Phase 12 planning** after RFC-0017 accepts.

---

## What this retrospective changes for Phase 12

- The plan-then-execute-with-checkpoints workflow stays.
- Subagent-driven-development was used for Phase 11 Tranche 11D — worked well for mechanical TDD-driven tasks; haiku for implementation, sonnet for the ship sequence (CI judgment).
- **New:** `docc-target: <Module>` in `.github/workflows/ci.yml` is now **mandatory from initial commit** for any package whose deps have DocC catalogs. Worth folding into `bare-swift new`'s scaffold so the trap is impossible to hit. Phase 12 follow-up: PR the scaffolder.
- **New:** Swift 6.3.1 typed-throws + `catch let e as ConcreteError` SILGen crash → drop the `as` cast; typed-throws guarantees the type. Memory-saved for any future executable / consumer code.
- **New:** DocC disambiguation hash suffixes (`-9rxiu` etc.) drift between Xcode docc and Linux docc. Use structural `->ReturnType` form or omit the suffix. Memory-saved.
- **New:** `Time.Duration` parameter types need explicit qualification; bare `Duration` collides with Swift stdlib `Duration`. Future packages depending on swift-time should qualify.

---

## Decision

Gate 11 is **PASSED on items 1, 2, 3, 4** and **NOT YET PASSED on item 5 (open)**. Phase 12 plans should be drafted but not executed until item 5 closes via RFC-0017.

Phase 11 closed the HTTP-server primitive stack — combined with Phases 4 / 7 / 8 / 9 / 10, the ecosystem now provides every primitive an HTTP server needs from wire decode through identity verification through wire encode. The auth-primitives wave shipped four packages in roughly one wall-clock day, matching RFC-0016's budget exactly.

The next step is **RFC-0017 anchoring Phase 12 on swift-brotli v0.2 encoder**, which both clears Gate 11 and closes the longest-standing v0.2 deferral in the compression tier. Internationalization, OTLP cross-signal, swift-jwt-verify v0.2 (signing), and OAuth 2.0 client primitives remain on the queue for Phase 13+.

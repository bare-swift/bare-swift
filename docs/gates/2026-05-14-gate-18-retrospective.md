# Gate 18 Retrospective: Phase 18 → Phase 19

**Date:** 2026-05-14
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 18 (the swift-distributed-tracing-bridge W3C bump anchored by [RFC-0023](../../rfcs/0023-phase-18-anchor-swift-distributed-tracing-bridge-w3c.md)) against the Gate 1–17 criteria template and recommends whether Phase 19 should start. Phase 18 was a **single-tranche additive minor bump** (18A swift-distributed-tracing-bridge v0.2). Shipped cleanly.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All anchored Tranche 18A deliverables shipped, CI green, DocC live, listed | ✓ PASS |
| 2 | RFC-0023 commitments fulfilled | ✓ PASS *(zero scope reshape; second consecutive clean RFC since Phase 17)* |
| 3 | ≥1 substantive RFC accepted *or documented absence of policy gaps* | ✓ PASS (documented absence of policy gaps) |
| 4 | RFC-0001 conventions stress-test against Phase 18 deliverables | ✓ DONE BELOW |
| 5 | Phase 19 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0024 below) |

**Overall:** Items 1–4 pass. Item 5 is open. Phase 19 plans should be drafted but not executed until item 5 closes via RFC-0024.

The roadmap's stop conditions did not trigger: Phase 18 calendar time was **<30 minutes wall-clock** — the smallest phase in the project to date. No security incidents.

---

## Item 1 — Phase 18 deliverables shipped ✓ PASS

| Tranche | Package | Tag | New code | DocC | Index | CI |
|---|---|---|---|---|---|---|
| 18A | swift-distributed-tracing-bridge | **v0.2.0** (existing-package minor bump) | ~30 source LOC + ~190 test LOC | ✓ | ✓ | green (first-try, push + tag) |

Single tranche shipped. **The Apple-frontend → bare-swift-OTLP adapter trinity is now production-deployable across HTTP boundaries:**
- `swift-distributed-tracing-bridge v0.2` (traces + W3C TraceContext extract/inject)
- `swift-log-bridge v0.1` (logs; auto-correlates via cascading `ServiceContext.otlpTraceIDs`)
- `swift-metrics-bridge v0.1` (metrics + exemplars; auto-correlates via cascading `ServiceContext.otlpTraceIDs`)

Cross-process trace correlation now works automatically: HTTP middleware that calls `InstrumentationSystem.instrument.extract / .inject` propagates W3C `traceparent` headers — and the log + metrics bridges auto-pick up the propagated traceIDs with **zero code changes** in those packages.

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists **48 packages** — no count change (existing-package minor bump).

**Calendar time:** <30 minutes wall-clock — smallest phase in the project. Bumped one dep version, replaced two method bodies, wrote 11 tests, updated CHANGELOG + README, shipped. The high value-to-LOC ratio matched RFC-0023's prediction.

**3-consecutive first-try-clean CI cycle:** Phase 16 (swift-log-bridge), Phase 17 (swift-metrics-bridge), Phase 18 (swift-distributed-tracing-bridge v0.2). Three phases in a row with zero CI fixup commits.

---

## Item 2 — RFC-0023 commitments ✓ PASS (zero scope reshape)

[RFC-0023](../../rfcs/0023-phase-18-anchor-swift-distributed-tracing-bridge-w3c.md) committed to:

| RFC-0023 commitment | Outcome |
|---|---|
| swift-distributed-tracing-bridge v0.2 (existing package, minor bump) | ✓ shipped. |
| Replace `Instrument.extract` no-op body with W3C parser | ✓ shipped. |
| Replace `Instrument.inject` no-op body with W3C serializer | ✓ shipped. |
| Use swift-tracing-otlp v0.3's already-shipped `OTLP.TraceContext.parse(traceparent:)` + `.traceparent` | ✓ shipped — zero re-implementation of W3C parsing. |
| Dep bump swift-tracing-otlp 0.1.0 → 0.3.0 | ✓ honored. |
| Non-breaking (v0.1 callers got no-ops; v0.2 callers get W3C behavior) | ✓ honored. |
| Single-tranche, ~150 LOC additive + ~15 tests, ~0.5 day calendar | ✓ shipped. Actual ~30 source LOC + ~190 test LOC (test count higher than estimate); ~30 min wall-clock (well under 0.5 day estimate). |
| Out of scope: B3 propagation, Jaeger, `tracestate`, HTTP middleware bindings | ✓ all honored. |
| Cascading benefit on log + metrics bridges (no code change) | ✓ confirmed. |

**Zero scope reshape this phase.** **Second consecutive clean-from-RFC phase** (Phase 17 was the first since Phase 13). The patterns introduced over Gates 14-16 (verify-before-drafting, brainstorm reality-check, primitives-pre-built-anticipating-integration) are now **stably preventing both rename and scope-cut churn**. The 5-instance reshape streak (Phases 12-16) appears to have given way to clean delivery as the procedural rules took hold.

**Notable execution efficiency:** Plan execution combined Tasks 2-6 (extract + inject + 11 tests across 5 logical tasks) into one inline session because the scope was so tight. The plan's bite-sized-task granularity was correct, but inline execution can collapse logically-adjacent tasks when the boundaries don't add value.

---

## Item 3 — Revised criterion (≥1 RFC accepted *or* documented absence of policy gaps) ✓ PASS

**Pre-Phase 18 (Gate 17 closeout):** RFC-0023 (Phase 18 anchor) accepted 2026-05-14.

**During Phase 18 execution:** zero RFCs accepted.

Phase 18 surfaced these findings during execution; all are memory-saved transferable lessons rather than cross-package policy gaps:

- **Primitives-pre-built-anticipating-integration pays off.** Phase 14B built `OTLP.TraceContext.parse(traceparent:)` and `.traceparent` accessor explicitly anticipating bridge integration. Phase 18 was integration only — ~30 LOC, zero re-implementation of W3C parsing, zero new test vectors (reusing swift-tracing-otlp's 13 parse tests). **Pattern:** when designing a value type that will need protocol integration later, build the parse + serialize surface in the value type's package, then the bridge package just wires the protocol.
- **Apple Instrument protocol shape verified against `.build/checkouts/`.** The brainstorm verified `Extractor.extract(key:from:)` + `Injector.inject(_:forKey:into:)` signatures by reading the actual Apple source in `.build/checkouts/swift-distributed-tracing/Sources/Instrumentation/Instrument.swift`. **Memory note:** when integrating with external protocols, grep `.build/checkouts/` for the actual signature instead of inferring from docs.
- **Single-tranche minor bumps can ship in <30 minutes.** Phase 18's wall-clock is the floor. Future micro-bumps (B3 propagation, tracestate, etc.) can plausibly land in similar windows when scope is tight and primitives exist.

None warrant a new cross-package RFC. **Documented absence of policy gaps. Criterion satisfied.**

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

Per-package review (single existing-package minor bump):

| Convention | swift-distributed-tracing-bridge v0.2 |
|---|---|
| Module name PascalCase, no `Swift` prefix | ✓ DistributedTracingBridge (unchanged) |
| Apache-2.0 + LLVM exception SPDX | ✓ |
| NOTICE crediting upstream | ✓ (unchanged from v0.1; still credits swift-distributed-tracing + intra-ecosystem deps) |
| Swift 6.0+ tools version | ✓ |
| macOS 15 platform floor | ✓ (unchanged) |
| Sendable-clean by default | ✓ |
| Strict concurrency in CI | ✓ |
| Single public error enum, typed throws | ✓ DistributedTracingBridgeError (unchanged) |
| Public APIs Foundation-free | ✓ |
| README tagline + ≤30-line example | ✓ updated with cross-process propagation example |
| CHANGELOG with v0.2.0 entry per Keep-a-Changelog | ✓ |
| CI green on macOS + Linux | ✓ all 7 jobs first-try |
| DocC bundle | ✓ |
| Sanitizers ON (no large static data) | ✓ |
| Non-breaking minor bump pattern | ✓ (v0.1 no-op semantics replaced with real W3C behavior under same signatures) |

### Deviations and findings

**1. Smallest-ever phase shape proven.** Phase 18 (~30 min wall-clock, ~30 source LOC) sets a new project floor. Useful precedent for future micro-bumps: dep bump + 1-2 method body replacements + focused test suite + docs. Pattern works.

**2. Cascading-benefit-via-shared-state pattern validated.** Phase 18 changed only the bridge's `extract`/`inject` bodies; swift-log-bridge + swift-metrics-bridge **automatically** gained cross-process trace correlation through their existing reads of `ServiceContext.otlpTraceIDs`. This is the **second time** the trinity benefits from a cascading change without code edits in dependent packages (the first was Phase 17 swift-metrics-bridge gaining exemplar correlation via the same TaskLocal). Pattern: shared TaskLocal state across the bridge family means upstream bridge changes propagate downstream automatically.

**3. `.build/checkouts/` grep is the canonical way to verify external-package protocol shapes.** Used in Phase 18 to confirm Apple Instrument signatures before writing the implementation. Reliable, fast, no docs-rot.

**4. Plan task granularity vs inline execution coalescing.** The plan had 6 tasks for the implementation (T1 dep bump, T2 failing test, T3 extract impl, T4 reject tests, T5 inject impl, T6 inject + roundtrip + chain). Inline execution naturally coalesced T2-T6 into one commit because the changes were so tight. The plan's bite-sized granularity is still correct for **subagent-driven** execution (clear hand-offs); for **inline execution on small bumps**, coalescing is fine. Memory note: tune granularity to execution mode.

**5. Single-Suite test pattern.** All 11 tests landed in one `W3CTraceContextTests` suite rather than splitting into Extract/Inject/RoundTrip suites. For 11 tests with shared fixtures (canonical traceparent + fake Extractor/Injector), one suite is right; only split when fixtures diverge.

### Verdict

RFC-0001 held cleanly under Phase 18 stress. The non-breaking minor bump pattern + cascading-benefit pattern are now production-validated.

---

## Item 5 — Phase 19 anchor decision ✗ NOT YET (recommendation)

Phase 19 anchor candidates surveyed (drawn from RFC-0023's "Sets up Phase 19+ options"):

| Wave | Packages | Risk | Adoption fit |
|---|---|---|---|
| **swift-jwt-verify v0.2 (signing — HS256 + ES256)** | swift-jwt-verify v0.2 | Medium-high | **High** — closes the verify-only design boundary; **8th rejection if deferred again**; v0.1 has had 2-4 days of settle time across the trinity-focus phases. |
| swift-distributed-tracing-bridge v0.3 (B3 + Jaeger propagation) | bridge v0.3 | Low-medium | Low — no concrete adopter demand; W3C just landed; would be 3rd v0.x bump on bridge in two phases. |
| swift-idna v0.3 (Bidi + ContextJ + ContextO) | swift-idna v0.3 | Medium | Medium — closes Phase 15's documented deferral. |
| swift-brotli v0.3 (streaming encoder) | swift-brotli v0.3 | Medium | Medium — closes Phase 12 v0.2 deferral. |
| OAuth 2.0 client primitives | swift-oauth2-client | Medium | Low-medium — no concrete adopter demand. |
| Crypto-adjacent | various | Medium-low | Low — 15th rejection candidate. |
| swift-publicsuffix v0.2 (ICANN/PRIVATE split) | swift-publicsuffix v0.2 | Low | Low — quiet patch when adopter asks. |

### Recommendation: **Anchor on swift-jwt-verify v0.2 (signing — HS256 + ES256)**

Reasons:

1. **8th rejection candidate this gate cycle.** swift-jwt-verify v0.2 signing has been deferred at every gate since Phase 11 closed (Gates 11-17, 7 prior deferrals). The "wait a week" criterion was reasonable when v0.1 was fresh; v0.1 has now had **2-4 days** of settle time across the trinity-focus phases, with no reported issues. The procedural deferral is wearing thin.
2. **Closes the longest-standing functional gap in the auth tier.** swift-jwt-verify v0.1 (Phase 11C) verifies signatures from issuers. swift-jwt-verify v0.2 will sign — making the package the full JWT primitive instead of verify-only. swift-bearer + swift-basic-auth handle the consumer side; signing closes the issuer side.
3. **Algorithm scope matches v0.1's verification surface.** v0.1 verifies HS256 + ES256. v0.2 signs the same two algorithms. **No new algorithm surface area** — the swift-crypto integrations (HMAC-SHA-256, P-256 ECDSA) are already in the package; v0.2 just exposes signing entry points alongside the existing verify entry points.
4. **Additive only.** Signing functions are new; verification functions are unchanged. **Non-breaking minor bump** — same shape as Phase 18.
5. **Audience continuity.** Auth/security adopters from Phase 11 (swift-basic-auth + swift-bearer + swift-jwt-verify). Net new community contact (vs. the observability trinity focus of Phases 14-18).
6. **Risk profile is well-bounded.**
   - swift-crypto provides `HMAC<SHA256>` and `P256.Signing` — both used by v0.1 verification; v0.2 signing reuses the same types in reverse direction (`init(authenticating:using:)` for HMAC, `signature(for:)` for ECDSA).
   - Test vectors: RFC 7515 JWS Appendix A has signing examples for HS256 + ES256. Easy round-trip: sign → verify → assert.
   - No new dependencies. No new external integrations.
7. **Sets up Phase 20+ options.**
   - **swift-distributed-tracing-bridge v0.3 (B3 + Jaeger)** stays available when adopter demand surfaces.
   - **swift-idna v0.3 (Bidi + ContextJ)** stays available.
   - **swift-brotli v0.3 streaming** stays available.
   - **OAuth 2.0 client** becomes more relevant once swift-jwt-verify can both sign and verify (OAuth flows often need both).
   - **RS256 (RSA)** is the natural v0.3 algorithm-expansion candidate.

### Phase 19 tranche sketch (subject to formal RFC)

Single-anchor phase shape, two reasonable decompositions:

**Shape A (recommended): single tranche, HS256 + ES256 signing matching v0.1's verify surface**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 19A | swift-jwt-verify v0.2 | `JWTSigner.signHS256(_:key:)` + `JWTSigner.signES256(_:key:)` + JWS Compact serialization + 12-15 tests | ~0.5-1 day |

**Shape B (alternate): two tranches if EC key handling expands**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 19A | swift-jwt-verify v0.2 | HS256 signing only | ~0.5 day |
| 19B | swift-jwt-verify v0.3 | ES256 signing (if EC scope larger than expected) | ~0.5 day |

Per the now-canonical brainstorm-phase reality-check pattern, expect Shape B if ES256 key marshalling (P-256 public/private keys, PKCS#8 / SEC1 encoding) exceeds the budget. swift-crypto's `P256.Signing.PrivateKey` should be sufficient surface, but the brainstorm should verify.

### Why other waves were rejected

- **swift-distributed-tracing-bridge v0.3 (B3 + Jaeger)** — W3C just landed in v0.2; third v0.x bump on the bridge in two phases would be aggressive churn. No concrete adopter demand for B3/Jaeger. Phase 20+ candidate.
- **swift-idna v0.3 (Bidi + ContextJ)** — third v0.x bump on the same package in four phases; rarely-hit rules; no adopter demand surfacing.
- **swift-brotli v0.3 streaming** — narrower audience than auth completion. Phase 20+ candidate.
- **OAuth 2.0** — still no concrete demand; would compose better after JWT signing lands.
- **Crypto-adjacent** — 15th consecutive rejection.
- **swift-publicsuffix v0.2** — narrow; quiet patch when adopter asks.

### Phase 18-19 boundary observation

Phase 18 was the project's smallest phase (<30 min). Phase 19 will be medium-sized (~0.5-1 day for HS256 + ES256 signing). The wall-clock cadence flexes with scope; the procedural cadence (RFC → brainstorm → plan → execute → gate) stays the same regardless. Pattern is now production-validated for micro (Phase 18), small (Phase 16: 1 day, ~245 LOC), medium (Phases 14C/15B/17A: 1 day, ~50-590 LOC), and large (Phase 15A: 1 day, ~680 LOC + 410 KiB data) phases.

**Action:** author the anchor decision as **RFC-0024** (Phase 19 anchor: swift-jwt-verify v0.2 signing) and accept it before Phase 19 plans start.

---

## Open work to clear Gate 18

1. **Write RFC-0024** (Phase 19 anchor: swift-jwt-verify v0.2 signing — HS256 + ES256). 1-2 hour task. References this retrospective.
2. **Begin Phase 19 planning** after RFC-0024 accepts. Brainstorm should verify swift-crypto's `HMAC<SHA256>` + `P256.Signing.PrivateKey` signing surface against the JWS Compact format expected by v0.1's verifier.

---

## What this retrospective changes for Phase 19

- The plan-then-execute-with-checkpoints workflow stays. Validated across micro / small / medium / large phase shapes.
- **Reinforced (firmly canonical now):** Primitives-pre-built-anticipating-integration pattern. Phase 14B's `OTLP.TraceContext.parse / .traceparent` design specifically anticipated bridge integration — Phase 18 delivered without re-implementing W3C parsing. Future RFCs that introduce value types should think about future protocol integrations.
- **Reinforced (firmly canonical now):** Cascading-benefit-via-shared-state pattern. The trinity's `ServiceContext.otlpTraceIDs` TaskLocal enables upstream bridge changes to propagate downstream automatically. Phase 18 confirmed this for the second time.
- **New observation:** Plan task granularity should adapt to execution mode. Bite-sized granularity is right for subagent-driven; inline execution can coalesce logically-adjacent tasks when the changes are tight.
- **New memory note:** `.build/checkouts/` grep is the canonical way to verify external-package protocol shapes during brainstorm.
- **Carried forward:** Phase 18 left no documented deferrals. The trinity is feature-complete in v0.2/v0.1/v0.1 (traces/logs/metrics).

---

## Decision

Gate 18 is **PASSED on items 1, 2, 3, 4** and **NOT YET PASSED on item 5 (open)**. Phase 19 plans should be drafted but not executed until item 5 closes via RFC-0024.

Phase 18 closed the cross-process correlation gap on the trinity — adopters can now bootstrap once and get fully OTLP-encoded + W3C-propagated observability across HTTP boundaries with standard Apple swift-distributed-tracing + swift-log + swift-metrics APIs.

The next step is **RFC-0024 anchoring Phase 19 on swift-jwt-verify v0.2 (signing — HS256 + ES256)**, which closes the verify-only design boundary in the auth tier after 7 prior gate-rejection deferrals. swift-distributed-tracing-bridge v0.3 (B3 + Jaeger), swift-idna v0.3, swift-brotli v0.3 streaming, RS256, and OAuth 2.0 remain on the queue for Phase 20+.

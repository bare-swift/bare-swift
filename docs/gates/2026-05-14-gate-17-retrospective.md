# Gate 17 Retrospective: Phase 17 → Phase 18

**Date:** 2026-05-14
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 17 (the swift-metrics-bridge wave anchored by [RFC-0022](../../rfcs/0022-phase-17-anchor-swift-metrics-bridge.md)) against the Gate 1–16 criteria template and recommends whether Phase 18 should start. Phase 17 was a **single-tranche wave** (17A swift-metrics-bridge v0.1). Shipped cleanly.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All anchored Tranche 17A deliverables shipped, CI green, DocC live, listed | ✓ PASS |
| 2 | RFC-0022 commitments fulfilled | ✓ PASS *(no scope reshape — first since Phase 15)* |
| 3 | ≥1 substantive RFC accepted *or documented absence of policy gaps* | ✓ PASS (documented absence of policy gaps) |
| 4 | RFC-0001 conventions stress-test against Phase 17 deliverables | ✓ DONE BELOW |
| 5 | Phase 18 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0023 below) |

**Overall:** Items 1–4 pass. Item 5 is open. Phase 18 plans should be drafted but not executed until item 5 closes via RFC-0023.

The roadmap's stop conditions did not trigger: Phase 17 calendar time was ~1 day wall-clock (matching RFC-0022's 1.5-day Shape A estimate). No security incidents. No Apple announcements that subsumed planned packages.

---

## Item 1 — Phase 17 deliverables shipped ✓ PASS

| Tranche | Package | Tag | New code | DocC | Index | CI |
|---|---|---|---|---|---|---|
| 17A | swift-metrics-bridge | **v0.1.0** (NEW) | ~590 LOC source + ~600 LOC tests | ✓ | ✓ | green (first-try, push + tag) |

Single tranche shipped. **The Apple-frontend → bare-swift-OTLP adapter trinity is now COMPLETE:**
- **swift-distributed-tracing-bridge v0.1** (Phase 3B) — Apple swift-distributed-tracing → OTLP traces
- **swift-log-bridge v0.1** (Phase 16A) — Apple swift-log → OTLP logs with auto trace correlation
- **swift-metrics-bridge v0.1** (Phase 17A) — Apple swift-metrics → OTLP metrics with auto exemplar correlation

End-to-end story: bootstrap all three factories once; standard Apple swift-distributed-tracing + swift-log + swift-metrics APIs produce OTLP-encoded traces + logs + metrics with cross-signal correlation. Trace IDs auto-attach to log records (via ServiceContext.otlpTraceIDs) AND metric exemplars (via the same TaskLocal).

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists **48 packages** — 1 new from Phase 17.

**Calendar time:** Phase 17 executed in ~1 day wall-clock (under RFC-0022's 1.5-day estimate). Repo scaffold + 6 handler classes + factory + 36 tests + docs + ship + umbrella + memory closure all in one session.

**Second consecutive new-package phase to ship with zero Pages-related fixup commits.** Pre-emptive Pages setup (three-API-call sequence codified in Gate 16) is now firmly canonical for new repos.

---

## Item 2 — RFC-0022 commitments ✓ PASS (no scope reshape)

[RFC-0022](../../rfcs/0022-phase-17-anchor-swift-metrics-bridge.md) committed to:

| RFC-0022 commitment | Outcome |
|---|---|
| New package `swift-metrics-bridge`, module `MetricsBridge` | ✓ shipped. |
| `OTLPMetricsFactory` conforming to Apple `MetricsFactory` | ✓ shipped. |
| 6 handler classes: Counter, FloatingPointCounter, HistogramRecorder, GaugeRecorder, Timer, Meter | ✓ all 6 shipped. |
| `flushExport() -> Bytes` + `takeBufferedMetrics() -> [OTLP.Metric]` + `registeredHandlerCount` | ✓ shipped. |
| Cross-signal exemplar auto-attach via `ServiceContext.otlpTraceIDs` | ✓ shipped (most-recent-only, per anti-goal). |
| `Configuration` with separate histogram + timer bucket defaults | ✓ shipped. |
| `MetricsBridgeError` typed-throws (empty for v0.1) | ✓ shipped. |
| Shape A (single tranche, full v0.1) | ✓ honored. |
| ~300-400 LOC estimate | ✗ actual ~590 source LOC — under-estimated but within Shape A's 1.5-day budget. |
| 6 direct deps | ✓ honored. |
| Out of scope: gRPC, HTTP, delta temporality, reservoir, periodic auto-flush, per-metric buckets, aggregation views | ✓ all honored. |
| Pre-emptive Pages setup per three-API-call sequence | ✓ honored (worked first-try). |
| Single-anchor phase | ✓ honored. |

**Zero scope reshape this phase** — first time since Phase 14 where the RFC's optimistic scope landed without rename or scope-cut. The verify-before-drafting procedural rule (Gate 14) + brainstorm-phase reality-check (4-for-4 prior pattern) **both** held cleanly: RFC-0022 named the package correctly the first time and the scope held under brainstorm scrutiny. **Pattern is now firmly stable.**

The LOC under-estimate (~300-400 vs actual ~590) is acceptable noise — the spec correctly identified 6 handler classes plus factory, and each handler is ~45-60 LOC including Mutex state + protocol methods + snapshot + exemplar capture. Future RFCs that propose N handler classes should budget ~50 LOC each plus ~150 LOC factory.

---

## Item 3 — Revised criterion (≥1 RFC accepted *or* documented absence of policy gaps) ✓ PASS

**Pre-Phase 17 (Gate 16 closeout):** RFC-0022 (Phase 17 anchor) accepted 2026-05-13.

**During Phase 17 execution:** zero RFCs accepted.

Phase 17 surfaced these findings during execution; all are memory-saved transferable lessons rather than cross-package policy gaps:

- **`@usableFromInline` + protocol-conformance friction.** Initially marked handler classes as `@usableFromInline internal final class` with `@usableFromInline internal func` on protocol-satisfying methods. Swift's rules: each protocol-conforming method MUST have `@usableFromInline` if the conforming type is `@usableFromInline` AND the protocol has any `@usableFromInline` requirements. Mixing was messier than expected. **Resolution:** stripped `@usableFromInline` entirely from the handler classes + methods. Plain `internal` is sufficient since the only external reference (OTLPMetricsFactory) is also same-module. **Lesson:** when conforming to a protocol with an internal class, either fully `@usableFromInline` everything or keep everything plain `internal`. Don't mix.
- **Pre-emptive Pages setup is now 2-for-2.** Phase 16 + Phase 17 both shipped Pages docs on first tag without fixup commits. Pattern is firmly canonical.
- **Bridge naming convention now has 3 instances.** swift-distributed-tracing-bridge + swift-log-bridge + swift-metrics-bridge. Future Apple-frontend → bare-swift-backend adapters should follow `swift-X-bridge`. Codified.

None warrant a new cross-package RFC. **Documented absence of policy gaps. Criterion satisfied.**

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

Per-package review (single new package):

| Convention | swift-metrics-bridge |
|---|---|
| Module name PascalCase, no `Swift` prefix | ✓ MetricsBridge |
| Apache-2.0 + LLVM exception SPDX | ✓ |
| NOTICE crediting upstream | ✓ swift-metrics + swift-distributed-tracing (Apple) + 2 bare-swift deps explicitly + transitives mentioned |
| Swift 6.0+ tools version | ✓ |
| macOS 15 platform floor | ✓ (matches Phase 16's swift-log-bridge; required by swift-distributed-tracing 1.x) |
| Sendable-clean by default | ✓ (final class @unchecked Sendable wrapping Mutex) |
| Strict concurrency in CI | ✓ |
| Single public error enum, typed throws | ✓ MetricsBridgeError (empty; forward-compat) |
| Public APIs Foundation-free | ✓ (defaultWallClock uses Darwin/Glibc clock_gettime directly) |
| Repo skeleton matches bare-swift conventions | ✓ (modeled on swift-log-bridge) |
| README tagline + ≤30-line example | ✓ |
| CHANGELOG with v0.x entry per Keep-a-Changelog | ✓ |
| CI green on macOS + Linux | ✓ (matrix: macOS 15 + ubuntu 22.04/24.04 × Swift 6.0/6.1) |
| DocC bundle | ✓ |
| docc-target set from initial commit (Gate 11 lesson) | ✓ |
| Sanitizers ON (no large static data) | ✓ |
| Pre-emptive Pages setup (Gate 16 codification) | ✓ — 2nd application |

### Deviations and findings

**1. Bridge naming convention at 3 instances.** swift-distributed-tracing-bridge + swift-log-bridge + swift-metrics-bridge. **Convention is now firmly canonical** for Apple-frontend → bare-swift-OTLP-backend adapters. Future bridges (e.g., a hypothetical swift-tracing-otlp / OAuth-related bridge) should follow `swift-X-bridge`.

**2. Adapter trinity completion is a milestone.** Phase 17 closes the observability adapter family. The Apple swift-distributed-tracing + swift-log + swift-metrics frontends all now have bare-swift OTLP backends with cross-signal correlation. This is the longest sustained focus area in the project: traces shipped in Phase 3 (2026-05-09), logs in Phase 16 (2026-05-13), metrics in Phase 17 (2026-05-14). Five days of focused observability work.

**3. Mutex-per-handler pattern continues to scale.** Phase 16's swift-log-bridge proved `Mutex<State>` per handler is the right shape. Phase 17's 6 handler classes confirm: low contention per handler, clean state isolation, no swift-atomics dep needed. Pattern is canonical for handler-class registries.

**4. Plan execution validated for medium-sized new-package phases.** Phase 16 (~245 LOC) and Phase 17 (~590 LOC) both ran cleanly inline via superpowers:writing-plans + executing-plans. The "one task per logical commit boundary" decomposition pattern scales from ~5 to ~16 tasks without breaking.

**5. `@usableFromInline` friction surfaced during implementation.** As described in Item 3 — when internal classes conform to public protocols, fully-strip-or-fully-apply is the right approach. Codified.

### Verdict

RFC-0001 held cleanly under Phase 17 stress. The bridge naming convention + cross-signal correlation pattern + pre-emptive Pages setup are now production-validated across multiple phases.

---

## Item 5 — Phase 18 anchor decision ✗ NOT YET (recommendation)

Phase 18 anchor candidates surveyed (drawn from RFC-0022's "Sets up Phase 18+ options"):

| Wave | Packages | Risk | Adoption fit |
|---|---|---|---|
| **swift-distributed-tracing-bridge v0.2 (W3C TraceContext extract/inject)** | bridge v0.2 | Low-medium | **High** — completes cross-process trace propagation for the fresh trinity; uses swift-tracing-otlp v0.3's already-shipped `OTLP.TraceContext.parse(traceparent:)` + `.traceparent` accessor. Production deployments need this. |
| swift-jwt-verify v0.2 signing (RS256/ES384/EdDSA) | swift-jwt-verify v0.2 | Medium-high | Medium — closes verify-only design boundary; v0.1 has had ~2 days to settle. |
| swift-idna v0.3 (Bidi + ContextJ + ContextO) | swift-idna v0.3 | Medium | Medium — closes Phase 15's documented deferral. |
| swift-brotli v0.3 (streaming encoder) | swift-brotli v0.3 | Medium | Medium — narrower than W3C propagation. |
| OAuth 2.0 client primitives | swift-oauth2-client | Medium | Low-medium — no concrete adopter demand. |
| Crypto-adjacent | various | Medium-low | Low — 14th rejection candidate. |
| swift-publicsuffix v0.2 (ICANN/PRIVATE split) | swift-publicsuffix v0.2 | Low | Low — narrow; better as patch. |

### Recommendation: **Anchor on swift-distributed-tracing-bridge v0.2 (W3C TraceContext extract/inject)**

Reasons:

1. **Closes the longest-standing documented deferral in the bridge.** swift-distributed-tracing-bridge v0.1's README explicitly notes "W3C TraceContext / B3 / Jaeger propagation are deferred to v0.2." Phase 14 added `OTLP.TraceContext` (W3C value type) to swift-tracing-otlp v0.3. Phase 18 wires it into the bridge's `Instrument.extract` / `Instrument.inject`.
2. **High value with the fresh trinity.** Phase 17 just shipped the observability adapter trinity. **The trinity works in-process but not across HTTP boundaries today** — cross-process correlation requires manual `traceparent` header plumbing in every adopter's HTTP middleware. Phase 18 closes this immediately while the trinity is fresh in adopters' minds.
3. **All primitives already exist.** swift-tracing-otlp v0.3 (Phase 14B) ships:
   - `OTLP.TraceContext.parse(traceparent:) -> TraceContext?` — strict W3C parser
   - `OTLP.TraceContext.traceparent: String?` — W3C header serializer
   These were specifically built anticipating bridge integration. The bridge just wires them into the Instrument protocol's `extract<Extract: Extractor>` and `inject<Inject: Injector>` methods.
4. **Audience continuity.** Same trinity adopters from Phase 17.
5. **Low risk.**
   - `extract`: read the "traceparent" header from the Carrier via the Extractor, call `OTLP.TraceContext.parse(traceparent:)`, populate `ServiceContext.otlpTraceIDs` from the returned TraceContext. ~30 LOC.
   - `inject`: read `ServiceContext.otlpTraceIDs`, construct an `OTLP.TraceContext`, get `.traceparent`, write to the Carrier via the Injector. ~30 LOC.
   - Both new code paths replace the v0.1 no-op stubs.
6. **Single-anchor phase, small scope.** ~150 LOC additive in swift-distributed-tracing-bridge + ~15 tests. Single tranche, ~0.5 day calendar.
7. **Sets up Phase 19+ options.**
   - **swift-jwt-verify v0.2 signing** has had a full week to settle.
   - **swift-idna v0.3** stays available.
   - **swift-brotli v0.3 streaming** stays available.
   - **OAuth 2.0** stays available.
   - **B3 + Jaeger propagation** become natural v0.3 follow-ons if adopter demand surfaces.

### Phase 18 tranche sketch (subject to formal RFC)

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 18A | swift-distributed-tracing-bridge v0.2 | W3C TraceContext extract/inject via swift-tracing-otlp v0.3's already-shipped types | ~0.5 day |

Single-tranche shape. Estimated ~150 LOC source + ~15 tests. Smaller than Phase 16-17, but the value-to-LOC ratio is exceptional since it unlocks cross-process correlation for the entire trinity with no additional package surface.

### Why other waves were rejected

- **swift-jwt-verify v0.2 signing** — v0.1 has had only ~2 days of adopter signal time. The "wait a week" criterion from RFC-0022 hasn't lapsed yet. Phase 19+ when v0.1 has had a full week of settle time.
- **swift-idna v0.3 (Bidi + ContextJ)** — three consecutive v0.x bumps on the same package would still be aggressive. Bidi+ContextJ rarely-hit. Phase 19+ when demand surfaces.
- **swift-brotli v0.3 streaming** — narrower audience than W3C propagation. Phase 19+ candidate.
- **OAuth 2.0** — still no concrete demand.
- **Crypto-adjacent** — 14th consecutive rejection.
- **swift-publicsuffix v0.2** — narrow; quiet patch when an adopter asks.

**Action:** author the anchor decision as **RFC-0023** (Phase 18 anchor: swift-distributed-tracing-bridge v0.2 W3C TraceContext) and accept it before Phase 18 plans start.

---

## Open work to clear Gate 17

1. **Write RFC-0023** (Phase 18 anchor: swift-distributed-tracing-bridge v0.2). 1-hour task — RFC-0023 is small because the scope is small and all primitives exist. References this retrospective.
2. **Begin Phase 18 planning** after RFC-0023 accepts.

---

## What this retrospective changes for Phase 18

- The plan-then-execute-with-checkpoints workflow stays. Validated for net-new packages (16-17) AND additive v0.x bumps (14B/14C/15A/15B). Phase 18 returns to additive-bump shape; pattern proven for both modes.
- **Reinforced (firmly canonical now):** Pre-emptive GitHub Pages setup is 2-for-2 (Phase 16 + Phase 17). Three-API-call sequence codified.
- **Reinforced (firmly canonical now):** Bridge naming convention `swift-X-bridge` at 3 instances.
- **New observation:** Phase 18 is the **first non-reshape phase since Phase 13**. Phase 14 reshaped (encoder packages got explicit-input APIs vs RFC-0019's implicit-context sketch). Phase 15 had two BREAKING bumps with documented migration. Phase 16 reshaped (RFC-0021 rename + scope-cut). Phase 17 was the first clean-from-RFC phase. Phase 18 will be the second — both small additive bumps. Pattern: **after the "open new direction" phase reshapes, subsequent same-direction phases land cleanly.**
- **New memory note:** `@usableFromInline` + protocol-conformance friction. Pick fully-`@usableFromInline` or fully-`internal`; don't mix.
- **Carried forward:** Phase 17 left no documented deferrals. The observability adapter trinity is complete.

---

## Decision

Gate 17 is **PASSED on items 1, 2, 3, 4** and **NOT YET PASSED on item 5 (open)**. Phase 18 plans should be drafted but not executed until item 5 closes via RFC-0023.

Phase 17 completed the Apple-frontend → bare-swift-OTLP adapter trinity. After ~5 days of focused observability work (traces Phase 3, logs Phase 16, metrics Phase 17), adopters can bootstrap once and emit OTLP-encoded traces + logs + metrics from standard Apple swift-distributed-tracing + swift-log + swift-metrics APIs with cross-signal correlation.

The next step is **RFC-0023 anchoring Phase 18 on swift-distributed-tracing-bridge v0.2 (W3C TraceContext extract/inject)**, which closes the in-process-only limitation of the trinity by adding cross-process trace propagation via the W3C `traceparent` header. swift-jwt-verify v0.2 signing, swift-idna v0.3, swift-brotli v0.3 streaming, and OAuth 2.0 remain on the queue for Phase 19+.

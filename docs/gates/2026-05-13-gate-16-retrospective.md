# Gate 16 Retrospective: Phase 16 → Phase 17

**Date:** 2026-05-13
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 16 (the swift-log-bridge wave anchored by [RFC-0021](../../rfcs/0021-phase-16-anchor-swift-distributed-tracing-otlp.md), reshaped during brainstorm) against the Gate 1–15 criteria template and recommends whether Phase 17 should start. Phase 16 was a **single-tranche wave** (16A swift-log-bridge v0.1). Shipped cleanly.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All anchored Tranche 16A deliverables shipped, CI green, DocC live, listed | ✓ PASS |
| 2 | RFC-0021 commitments fulfilled (reshaped during brainstorm) | ✓ PASS *(documented rename + scope-cut)* |
| 3 | ≥1 substantive RFC accepted *or documented absence of policy gaps* | ✓ PASS (documented absence of policy gaps) |
| 4 | RFC-0001 conventions stress-test against Phase 16 deliverables | ✓ DONE BELOW |
| 5 | Phase 17 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0022 below) |

**Overall:** Items 1–4 pass. Item 5 is open. Phase 17 plans should be drafted but not executed until item 5 closes via RFC-0022.

The roadmap's stop conditions did not trigger: Phase 16 calendar time was ~1 day wall-clock (matching RFC-0021's revised 1-1.5-day estimate). No security incidents. No Apple announcements that subsumed planned packages.

---

## Item 1 — Phase 16 deliverables shipped ✓ PASS

| Tranche | Package | Tag | New code | DocC | Index | CI |
|---|---|---|---|---|---|---|
| 16A | swift-log-bridge | **v0.1.0** (NEW) | ~245 LOC source + ~400 LOC tests | ✓ | ✓ | green (first-try, push + tag) |

Single tranche shipped. The observability adapter family is now two-of-three complete:
- **swift-distributed-tracing-bridge v0.1** (Phase 3B) — Apple swift-distributed-tracing → OTLP traces.
- **swift-log-bridge v0.1** (Phase 16A, **this**) — Apple swift-log → OTLP logs with auto trace correlation.
- _Phase 17+ candidate:_ swift-metrics-bridge — Apple swift-metrics → OTLP metrics (closes the observability trinity).

End-to-end story now usable: bootstrap `OTLPTracer` + `OTLPLogHandler` once, then standard `withSpan { logger.info(...) }` produces records that auto-carry `traceID` / `spanID`.

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists **47 packages** — 1 new from Phase 16.

**Calendar time:** Phase 16 executed in ~1 day wall-clock (under RFC-0021's 1-1.5-day estimate even after scope reshape). Pre-emptive GitHub Pages setup (per [[feedback-github-pages-new-repo]]) prevented the typical first-tag docs failure — **first new-repo phase to ship with zero Pages-related fixup commits.**

**Cleanest new-package shipping cycle to date.** First-try CI green on every workflow (push + tag-triggered Release + tag-triggered Publish docs). Zero fixup commits.

---

## Item 2 — RFC-0021 commitments ✓ PASS (with documented reshape)

[RFC-0021](../../rfcs/0021-phase-16-anchor-swift-distributed-tracing-otlp.md) anchored Phase 16 on a new adapter package called `swift-distributed-tracing-otlp` with three components: `OTLPLogHandler`, `OTLPTracer`, and `OTLP.TraceContext.current` accessor. Brainstorm phase surfaced that the existing `swift-distributed-tracing-bridge` v0.1 (Phase 3B) **already provides `OTLPTracer` + `OTLPTraceIDs` + `ServiceContext.otlpTraceIDs`**. The actual gap was just the log side.

**Brainstorm-phase reshape:**

| RFC-0021 commitment | Outcome |
|---|---|
| New package `swift-distributed-tracing-otlp` | ✗ **Renamed to `swift-log-bridge`** — matches the established `swift-distributed-tracing-bridge` "Apple frontend → bare-swift backend" naming convention. |
| Module `DistributedTracingOTLP` | ✗ **Renamed to `LogBridge`** — matches the package rename. |
| `OTLPTracer` (Tracer protocol implementation) | ✗ **Dropped — already exists in swift-distributed-tracing-bridge.** |
| `OTLPLogHandler` (LogHandler protocol implementation) | ✓ shipped. |
| `OTLP.TraceContext.current(in:)` accessor | ✓ shipped. |
| `DistributedTracingOTLPError` typed-throws | ✓ shipped as `LogBridgeError` (empty enum; forward-compat extension point). |
| 4 intra-ecosystem deps | ✓ honored — actual 5 deps (added swift-distributed-tracing-bridge for `OTLPTraceIDs`). |
| Single-anchor shape (Shape A) | ✓ honored — single tranche, ~245 LOC. |
| Out of scope: gRPC, HTTP client, sampler, batch/retry, cross-process propagation | ✓ all honored. |
| `bare-swift gen-site --umbrella .` after release | ✓ honored (umbrella bump in same calendar day). |
| Sanitizers ON (no large static data) | ✓ honored. |

**Doc-error note:** RFC-0021 explicitly anticipated this kind of reshape in its "brainstorm should verify the swift-distributed-tracing protocol shape against the reference implementation" caveat. The verification-against-reference pattern worked — the gap was caught during brainstorm, not during implementation. **Fourth instance** of the brainstorm-phase reality-check scope-cut pattern (after swift-brotli v0.2 encoder, swift-idna v0.1 Punycode-only, swift-otlp-json hand-rolled JSON writer). Pattern is now firmly canonical.

---

## Item 3 — Revised criterion (≥1 RFC accepted *or* documented absence of policy gaps) ✓ PASS

**Pre-Phase 16 (Gate 15 closeout):** RFC-0021 (Phase 16 anchor) accepted 2026-05-13.

**During Phase 16 execution:** zero RFCs accepted.

The revised criterion accepts "documented absence of policy gaps" as equivalent. Phase 16 surfaced these findings during execution; all are memory-saved transferable lessons rather than cross-package policy gaps:

- **swift-log Logger metadata is handler metadata.** Apple swift-log's `Logger.metadata` delegates directly to `handler.metadata`. Two `Logger` values constructed from the same factory share metadata. Memory note saved. **Lesson:** verify swift-log Logger vs handler semantics before asserting per-Logger isolation in tests.
- **Pre-emptive GitHub Pages setup pattern is now canonical.** Three API calls in sequence: (1) `gh api -X POST /repos/.../pages -f 'build_type=workflow'`; (2) `gh api -X PUT /repos/.../environments/github-pages` with `custom_branch_policies: true`; (3) `gh api -X POST /repos/.../environments/github-pages/deployment-branch-policies -f 'name=v*' -f 'type=tag'`. Result: zero Pages-related fixup commits on first tag. Reinforces [[feedback-github-pages-new-repo]] with concrete API commands.
- **Brainstorm-phase reality-check produces scope-cut.** Phase 16 is the fourth phase in a row where the RFC's optimistic scope was reality-checked against the existing codebase / reference implementations during brainstorm and resulted in a smaller v0.1. Pattern is now canonical.

None warrant a new cross-package RFC. **Documented absence of policy gaps. Criterion satisfied.**

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

Per-package review (single new package):

| Convention | swift-log-bridge |
|---|---|
| Module name PascalCase, no `Swift` prefix | ✓ LogBridge |
| Apache-2.0 + LLVM exception SPDX | ✓ |
| NOTICE crediting upstream | ✓ swift-log + swift-distributed-tracing (Apple) + 3 bare-swift deps |
| Swift 6.0+ tools version | ✓ |
| macOS 14 platform floor | ✓ (actually macOS 15 — matches swift-distributed-tracing-bridge; Apple swift-distributed-tracing 1.x requires it) |
| Sendable-clean by default | ✓ (final class @unchecked Sendable wrapping Mutex) |
| Strict concurrency in CI | ✓ |
| Single public error enum, typed throws | ✓ LogBridgeError (no cases yet; forward-compat) |
| Public APIs Foundation-free | ✓ (defaultWallClock uses Darwin/Glibc clock_gettime directly) |
| Repo skeleton matches bare-swift conventions | ✓ (modeled on swift-distributed-tracing-bridge) |
| README tagline + ≤30-line example | ✓ |
| CHANGELOG with v0.x entry per Keep-a-Changelog | ✓ |
| CI green on macOS + Linux | ✓ (5-platform matrix: macOS 15 + ubuntu 22.04/24.04 × Swift 6.0/6.1) |
| DocC bundle | ✓ |
| docc-target set from initial commit (Gate 11 lesson) | ✓ |
| Sanitizers ON (no large static data) | ✓ |
| Pre-emptive Pages setup (Gate 16 codification) | ✓ FIRST APPLICATION |

### Deviations and findings

**1. Apple-frontend → bare-swift-backend bridge naming convention now canonical.** Phase 3B's swift-distributed-tracing-bridge established the pattern. Phase 16A's swift-log-bridge confirms it. **Future bridge packages should follow:** swift-X-bridge for "Apple swift-X → bare-swift OTLP/output."

**2. Cross-package extension pattern.** swift-log-bridge extends `OTLP.TraceContext` (defined in swift-tracing-otlp) with a `.current(in:)` static accessor. This is the **first time** a bridge package extends a type from another bare-swift package's public API. Worked cleanly under DocC strict mode (the extension is used as prose only in the README, not as a DocC symbol link — per [[feedback-docc-cross-package]]).

**3. Deepest direct dep graph in the ecosystem.** swift-log-bridge depends on 5 bare-swift packages + 2 Apple packages. Pulls in ~10 packages transitively. SwiftPM resolved cleanly on first push. Strict-concurrency CI matrix (Ubuntu 22.04/24.04 × Swift 6.0/6.1 + macOS 15 × Swift 6.0) all green first-try.

**4. Pre-emptive Pages setup ships with zero fixup commits.** Phase 16 is the **first new-package phase to ship Pages docs on first tag** without a follow-up fixup commit. The pattern from [[feedback-github-pages-new-repo]] worked exactly as documented. Worth concretizing the API commands in that memory (already updated in 16A's memory file).

**5. Plan execution scaled to a fresh-repo flow.** Phase 16 was the first phase in this project to scaffold a brand-new repo end-to-end (vs the bumps and additions in Phases 14-15). The 14-task plan executed cleanly inline. Repo creation via `gh repo create`, Pages setup, three workflow files, six source/test files, ship-and-tag — all in one session.

### Verdict

RFC-0001 held cleanly under Phase 16 stress. The new bridge-naming convention is consistent and the pre-emptive Pages setup is now production-validated.

---

## Item 5 — Phase 17 anchor decision ✗ NOT YET (recommendation)

Phase 17 anchor candidates surveyed (drawn from RFC-0021's "Sets up Phase 17+ options" + new Phase 16-emerging candidates):

| Wave | Packages | Risk | Adoption fit |
|---|---|---|---|
| **swift-metrics-bridge (new package — completes observability trinity)** | swift-metrics-bridge v0.1 | Medium | **High** — closes the symmetric gap: Apple swift-metrics has no OTLP backend (swift-prometheus-metrics serves Prometheus only). Audience continuity with the swift-log-bridge cohort. |
| swift-jwt-verify v0.2 signing (RS256/ES384/EdDSA) | swift-jwt-verify v0.2 | Medium-high | Medium — closes verify-only design boundary; v0.1 has had ~2 days to settle. |
| swift-idna v0.3 (Bidi + ContextJ + ContextO) | swift-idna v0.3 | Medium | Medium — closes Phase 15's documented deferral; ~500 LOC. |
| swift-brotli v0.3 (streaming encoder) | swift-brotli v0.3 | Medium | Medium — closes Phase 12 v0.2 deferral; narrower audience. |
| swift-distributed-tracing-bridge v0.2 (W3C TraceContext extract/inject) | bridge v0.2 | Medium | Medium — would benefit swift-log-bridge auto-correlation but no cascading dependents yet. |
| OAuth 2.0 client primitives | swift-oauth2-client | Medium | Low-medium — no concrete adopter demand. |
| Crypto-adjacent | swift-blake3 / swift-siphash / swift-hkdf | Medium-low | Low — 13th rejection candidate. |
| swift-publicsuffix v0.2 (ICANN/PRIVATE split) | swift-publicsuffix v0.2 | Low | Low — narrow scope; better as quiet patch. |

### Recommendation: **Anchor on swift-metrics-bridge (new package — closes observability trinity)**

Reasons:

1. **Closes the symmetric Phase 16 gap.** Phase 16 shipped swift-log-bridge (Apple swift-log → OTLP logs). The observability adapter family now has 2-of-3:
   - swift-distributed-tracing-bridge (Apple swift-distributed-tracing → OTLP traces) — Phase 3B
   - swift-log-bridge (Apple swift-log → OTLP logs) — Phase 16A
   - **MISSING: swift-metrics-bridge** (Apple swift-metrics → OTLP metrics)
   The asymmetry is conspicuous. Phase 17 closes it.

2. **Concrete gap, not a design observation.** Adopters who want to export Apple swift-metrics as OTLP metrics today have **no bridge package**. The only swift-metrics adapter in the ecosystem is swift-prometheus-metrics (Prometheus output, not OTLP). Production observability stacks using OTLP for traces+logs (Phase 14-16) but Prometheus for metrics is a real pattern, but adopters increasingly want OTLP-everywhere.

3. **Audience continuity.** Same observability adopters who use swift-distributed-tracing-bridge + swift-log-bridge + swift-otlp-exporter. Tight composition story.

4. **Net-new package keeps SemVer churn manageable.** Phase 15 had two BREAKING bumps; Phase 16 was a new package (no breaking). Phase 17 staying on net-new packages continues to keep adopters' SemVer surface stable.

5. **Risk profile is well-bounded.**
   - swift-metrics has 3 metric types: Counter, Recorder (Gauge / Histogram), Timer. OTLP has corresponding types: Sum, Gauge, Histogram, ExponentialHistogram, Summary. Mapping is well-specified.
   - The `MetricsFactory` protocol is the entry point; each metric handler is a `final class` matching swift-prometheus-metrics's pattern exactly.
   - **Cross-signal exemplar correlation:** when `ServiceContext.current?.otlpTraceIDs` is active, metric data points can carry exemplars linking to the span. swift-otlp-exporter already has `OTLP.Exemplar` types.
   - Reference implementations in opentelemetry-swift.

6. **Single-anchor phase shape.** Like Phase 16A. Estimated ~300-400 LOC source + ~30 tests + ~1.5 days calendar.

7. **Sets up Phase 18+ options.**
   - **swift-jwt-verify v0.2 signing** has had a full week to settle.
   - **swift-distributed-tracing-bridge v0.2** (W3C extract/inject) becomes more valuable once all three bridges are in production.
   - **swift-idna v0.3 / swift-brotli v0.3** stay available.
   - **OAuth 2.0** stays available.

### Phase 17 tranche sketch (subject to formal RFC)

Single-anchor phase shape, two possible decompositions:

**Shape A (recommended): single tranche, full v0.1**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 17A | swift-metrics-bridge v0.1 | `OTLPMetricsFactory` (Apple `MetricsFactory` protocol implementation) + per-handler classes (Counter/Recorder/Timer) + buffer + `flushExport()` + cross-signal exemplar auto-attach via `ServiceContext.otlpTraceIDs` | ~1.5 days |

**Shape B (alternate): two tranches if exemplar correlation expands**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 17A | swift-metrics-bridge v0.1 | MetricsFactory + handlers + flushExport (no exemplars) | ~1 day |
| 17B | swift-metrics-bridge v0.2 | Cross-signal exemplar auto-attach | ~0.5 day |

Decision deferred to brainstorm phase (verify swift-metrics protocol shape against the reference). Per the now-canonical pattern, expect Shape B if exemplar wiring is wider than expected.

### Why other waves were rejected

- **swift-jwt-verify v0.2 signing** — v0.1 has had only ~2 days of adopter signal time. Phase 18+ when v0.1 has had a week.
- **swift-idna v0.3 (Bidi + ContextJ)** — third v0.x bump on same package in three phases would be aggressive churn. Bidi+ContextJ rarely-hit in real-world hostname data. Phase 18+ when demand surfaces.
- **swift-brotli v0.3 streaming** — narrower audience. Phase 18+ candidate.
- **swift-distributed-tracing-bridge v0.2 (W3C extract/inject)** — would benefit swift-log-bridge auto-correlation but no cascading demand yet. Phase 18+ candidate when the trinity is feature-complete on the Phase 17 metrics side.
- **OAuth 2.0** — still no concrete demand.
- **swift-publicsuffix v0.2** — narrow scope; better as patch.
- **Crypto-adjacent** — 13th consecutive rejection.

**Action:** author the anchor decision as **RFC-0022** (Phase 17 anchor: swift-metrics-bridge) and accept it before Phase 17 plans start.

---

## Open work to clear Gate 16

1. **Write RFC-0022** (Phase 17 anchor: swift-metrics-bridge). 1-2 hour task. References this retrospective.
2. **Begin Phase 17 planning** after RFC-0022 accepts. Brainstorm should verify the swift-metrics `MetricsFactory` protocol shape against the reference implementation (swift-prometheus-metrics is the in-house reference).

---

## What this retrospective changes for Phase 17

- The plan-then-execute-with-checkpoints workflow stays. Validated for fresh-repo flow now (Phase 16) in addition to existing-package bumps.
- **Reinforced (firmly canonical now):** Brainstorm-phase reality-check produces scope-cut. Pattern is at 4-for-4 ratio since it emerged in Phase 12. Future RFCs that propose new packages should expect brainstorm-phase rename / scope-cut and pre-allocate the time.
- **Reinforced + concretized:** Pre-emptive GitHub Pages setup ([[feedback-github-pages-new-repo]]) is now production-validated with the three-API-call sequence captured in 16A's memory.
- **New observation:** Apple-frontend → bare-swift-OTLP bridges form a natural family. With swift-distributed-tracing-bridge + swift-log-bridge done, swift-metrics-bridge is the symmetric closure. Future bridge packages should follow the `swift-X-bridge` naming convention.
- **Carried forward:** Phase 16 left no documented deferrals. The implicit-context bridge gap from Phase 14 is now closed for traces (Phase 3B) and logs (Phase 16A). Metrics is the next closure (Phase 17).

---

## Decision

Gate 16 is **PASSED on items 1, 2, 3, 4** and **NOT YET PASSED on item 5 (open)**. Phase 17 plans should be drafted but not executed until item 5 closes via RFC-0022.

Phase 16 closed the longest-standing observability-adapter gap on the log side. Adopters can now bootstrap once and get auto-correlated trace+log records via standard Apple swift-distributed-tracing + swift-log APIs.

The next step is **RFC-0022 anchoring Phase 17 on swift-metrics-bridge**, which closes the symmetric gap on the metrics side and completes the Apple-frontend → bare-swift-OTLP adapter trinity. swift-jwt-verify v0.2 signing, swift-idna v0.3, swift-brotli v0.3 streaming, swift-distributed-tracing-bridge v0.2 (W3C extract/inject), and OAuth 2.0 remain on the queue for Phase 18+.

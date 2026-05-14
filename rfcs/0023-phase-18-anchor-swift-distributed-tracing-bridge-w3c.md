# RFC-0023 — Phase 18 anchor: swift-distributed-tracing-bridge v0.2 (W3C TraceContext)

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-14 |
| Resolution | Accepted 2026-05-14 — Phase 18 plans may begin authoring. Single existing package: swift-distributed-tracing-bridge v0.2. |

## Summary

Anchor Phase 18 on **swift-distributed-tracing-bridge v0.2** — a minor version bump that wires W3C TraceContext extract/inject into Apple swift-distributed-tracing's `Instrument` protocol, using the already-shipped `OTLP.TraceContext.parse(traceparent:)` and `OTLP.TraceContext.traceparent` from swift-tracing-otlp v0.3. Closes the bridge's documented v0.2 deferral and unlocks cross-process correlation for the entire Apple-frontend → bare-swift-OTLP adapter trinity.

Single-tranche phase. Small scope (~150 LOC additive + ~15 tests). High value-to-LOC ratio: the trinity becomes production-deployable across HTTP boundaries with no additional package surface.

## Problem

[Gate 17 retrospective](../docs/gates/2026-05-14-gate-17-retrospective.md) item 5 requires "Phase 18 anchor decision recorded as an RFC" before Phase 18 plans begin. The retrospective surveyed seven candidate waves; this RFC formalizes the choice.

**Concrete gap:** Phase 17 completed the observability adapter trinity:
- swift-distributed-tracing-bridge (traces) — Phase 3B
- swift-log-bridge (logs) — Phase 16A
- swift-metrics-bridge (metrics) — Phase 17A

All three bridges work **in-process** but **not across HTTP boundaries**. Cross-process trace correlation requires the W3C `traceparent` header to be extracted from inbound HTTP requests and injected into outbound HTTP requests. Today, adopters using Apple swift-distributed-tracing + the bare-swift bridges have to **manually plumb traceparent headers in their HTTP middleware** at every service boundary — defeating much of the benefit of the trinity's bootstrap-once design.

The Apple `Instrument` protocol (which swift-distributed-tracing-bridge's `OTLPTracer` conforms to) defines exactly the right hooks:

```swift
func extract<Carrier, Extract: Extractor>(_ carrier: Carrier, into context: inout ServiceContext, using extractor: Extract) where Extract.Carrier == Carrier
func inject<Carrier, Inject: Injector>(_ context: ServiceContext, into carrier: inout Carrier, using injector: Inject) where Inject.Carrier == Carrier
```

In v0.1, both are no-ops with a documented comment: "v0.1: no W3C TraceContext / B3 / Jaeger extraction." Phase 18 wires them up.

**All primitives already exist.** swift-tracing-otlp v0.3 (Phase 14B) shipped:
- `OTLP.TraceContext.parse(traceparent:) -> TraceContext?` — strict W3C parser (rejects wrong length, missing dashes, non-`00` version, uppercase hex, all-zero IDs)
- `OTLP.TraceContext.traceparent: String?` — W3C header serializer

These were built anticipating bridge integration. Phase 18 is the integration.

After Phase 18, the trinity story becomes: bootstrap once at app startup → standard Apple APIs produce OTLP-encoded signals → HTTP middleware that uses `InstrumentationSystem.instrument.extract/inject` automatically propagates trace context across services. Zero per-call-site plumbing.

## Proposal

### Anchor: swift-distributed-tracing-bridge v0.2 (single existing package)

**swift-distributed-tracing-bridge v0.2** — wire W3C TraceContext into the `Instrument.extract` / `Instrument.inject` no-op stubs.

| API change | Description |
|---|---|
| `OTLPTracer.extract(_:into:using:)` | Read `"traceparent"` header from Carrier via Extractor → `OTLP.TraceContext.parse(traceparent:)` → if non-nil, populate `context.otlpTraceIDs` from the returned TraceContext's traceID + spanID. |
| `OTLPTracer.inject(_:into:using:)` | Read `context.otlpTraceIDs` → construct `OTLP.TraceContext(traceID:, spanID:, traceFlags: 0x01)` → get `.traceparent` String → write to Carrier via Injector under key `"traceparent"`. |
| `OTLPTracer.startSpan` | (unchanged in v0.2; already reads parent IDs from `context.otlpTraceIDs`) |

**Dependency bump:** swift-tracing-otlp 0.1.0 → 0.3.0 (already shipped; brings `OTLP.TraceContext` value type + parse/serialize).

**Additive only:** all v0.1 APIs unchanged. The two `Instrument` methods replace no-op bodies with real implementations; the method signatures don't change.

### Tranches

Single-tranche phase shape:

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 18A | swift-distributed-tracing-bridge v0.2 | W3C `traceparent` extract + inject via swift-tracing-otlp v0.3 primitives | ~0.5 day |

Smaller calendar than Phases 14-17. Reflects the focused scope: 2 method bodies + tests. No new modules, no new types, no architectural changes.

### Tranche 18A scope sketch

Concretely, in `Sources/DistributedTracingBridge/OTLPTracer.swift`, replace:

```swift
public func extract<Carrier, Extract: Extractor>(
    _ carrier: Carrier,
    into context: inout ServiceContext,
    using extractor: Extract
) where Extract.Carrier == Carrier {
    // v0.1: no W3C TraceContext / B3 / Jaeger extraction. ...
}
```

with:

```swift
public func extract<Carrier, Extract: Extractor>(
    _ carrier: Carrier,
    into context: inout ServiceContext,
    using extractor: Extract
) where Extract.Carrier == Carrier {
    guard let header = extractor.extract(key: "traceparent", from: carrier) else { return }
    guard let traceContext = OTLP.TraceContext.parse(traceparent: header) else { return }
    context.otlpTraceIDs = OTLPTraceIDs(
        traceID: traceContext.traceID,
        spanID: traceContext.spanID
    )
}
```

And similarly for `inject`:

```swift
public func inject<Carrier, Inject: Injector>(
    _ context: ServiceContext,
    into carrier: inout Carrier,
    using injector: Inject
) where Inject.Carrier == Carrier {
    guard let ids = context.otlpTraceIDs else { return }
    let tc = OTLP.TraceContext(
        traceID: ids.traceID,
        spanID: ids.spanID,
        traceFlags: 0x01
    )
    guard let header = tc.traceparent else { return }
    injector.inject(header, forKey: "traceparent", into: &carrier)
}
```

Plus tests covering:
- extract with valid traceparent populates ServiceContext.otlpTraceIDs
- extract with missing header is a no-op (context unchanged)
- extract with malformed traceparent is a no-op
- extract with uppercase hex is rejected (per W3C spec)
- extract with all-zero IDs is rejected
- inject with no active otlpTraceIDs is a no-op
- inject with valid otlpTraceIDs writes well-formed traceparent
- inject + extract round-trip preserves trace + span IDs
- Tests use a fake Extractor / Injector wrapping a `[String: String]` dict

### Why this anchor

1. **Closes the longest-standing v0.2 deferral on the trinity.** swift-distributed-tracing-bridge v0.1 shipped 2026-05-10 with W3C explicitly deferred to v0.2. Four days later, with the trinity fresh, this becomes high-value.
2. **All primitives exist; integration only.** Phase 14B built `OTLP.TraceContext.parse(traceparent:)` and `.traceparent` accessor specifically anticipating this integration. 13 tests on the parse/serialize side already.
3. **Cascading benefit to log + metrics bridges.** Both swift-log-bridge and swift-metrics-bridge already read `ServiceContext.otlpTraceIDs` for cross-signal correlation. Once the tracing bridge populates it from inbound HTTP headers via `extract`, **the log + metrics bridges automatically pick up cross-process traceIDs with no code changes.**
4. **Audience continuity.** Same trinity adopters from Phases 14, 16, 17.
5. **Risk profile is well-bounded.**
   - `OTLP.TraceContext.parse` has 13 strict-parsing tests already; we're not re-validating W3C parsing.
   - The Apple `Instrument` protocol is stable.
   - No new dependencies.
   - The change is additive — v0.1 callers using the no-op `extract` / `inject` got no behavior; v0.2 callers using them get the W3C behavior. **Not a breaking change** because the v0.1 contract was "no-op."
6. **Sets up Phase 19+ options.**
   - **swift-jwt-verify v0.2 signing** has had ~3-4 days of v0.1 settle time at minimum.
   - **B3 + Jaeger propagation** become natural v0.3 follow-ons if adopter demand surfaces.
   - **swift-idna v0.3 / swift-brotli v0.3 / OAuth 2.0** stay available.

### Out of scope for Phase 18

- **B3 propagation.** Different header format (`X-B3-TraceId` / `X-B3-SpanId` / `X-B3-Sampled`). Deferred to v0.3.
- **Jaeger propagation.** Different header format (`uber-trace-id`). Deferred to v0.3.
- **`tracestate` header.** Vendor-specific propagation header companion to `traceparent`. Already in `OTLP.TraceContext.traceState` field but v0.2 doesn't extract/inject it (round-trips at the value-type level only).
- **swift-jwt-verify v0.2 signing / swift-idna v0.3 / swift-brotli v0.3 / OAuth 2.0.** Phase 19+ candidates.

### Tooling expectations

Phase 18 reuses all Phase 7–17 tooling without extension:

- Existing swift-distributed-tracing-bridge repo (no new scaffolding).
- Strict-concurrency in CI stays mandatory.
- `docc-target` already set.
- Sanitizers ON (no large static data).
- Cross-library validation at ship time: round-trip extract→inject with the OTel collector reference.

### Versioning

| Package | New | Future |
|---|---|---|
| swift-distributed-tracing-bridge | v0.2 (W3C extract/inject) | v0.3 = B3 + Jaeger propagation if adopter demand; v0.4 = tracestate extract/inject |

Phase 18 introduces **0 new packages** (existing-package minor bump). Ecosystem stays at **48 packages**.

### Anti-goals

- **No B3 / Jaeger.** v0.2 is W3C TraceContext only.
- **No `tracestate` extract/inject.** Field is in the value type; v0.2 doesn't wire the header.
- **No new umbrella tier.**
- **No retroactive changes** to swift-tracing-otlp / swift-log-otlp / swift-otlp-exporter. v0.2 of the bridge is purely additive on top.
- **No change to existing v0.1 APIs.** Existing OTLPTracer initializer + extension methods unchanged.

## Alternatives considered

### Anchor Phase 18 on swift-jwt-verify v0.2 (signing + RS256/ES384/EdDSA)

**Pros:** rounds out the JWT story for issuers.

**Cons:** v0.1 (Phase 11) shipped 2026-05-12 — only ~2 days ago. RFC-0022 set the bar at "wait a week" for adopter signal. Multi-algorithm scope is wider than this single-tranche W3C integration. Phase 19+ when v0.1 has had a full week.

**Verdict:** rejected as Phase 18. Strong Phase 19 candidate.

### Anchor Phase 18 on swift-idna v0.3 (Bidi + ContextJ + ContextO)

**Pros:** closes Phase 15's documented deferral.

**Cons:** third v0.x bump on same package in four phases would still be aggressive. Bidi+ContextJ rarely-hit. Phase 19+ when demand surfaces.

**Verdict:** rejected as Phase 18.

### Anchor Phase 18 on swift-brotli v0.3 (streaming encoder)

**Pros:** closes Phase 12 v0.2 one-shot-only deferral.

**Cons:** narrower audience than W3C trinity propagation. Phase 19+ candidate.

**Verdict:** rejected as Phase 18.

### Anchor Phase 18 on OAuth 2.0 client primitives

**Pros:** composes naturally.

**Cons:** still no concrete adopter demand.

**Verdict:** rejected as Phase 18.

### Skip ahead, do crypto-adjacent (swift-blake3 / swift-siphash / swift-hkdf)

**Pros:** would complement swift-jwt-verify if signing ever lands.

**Cons:** swift-crypto remains entrenched. **14th consecutive rejection.**

**Verdict:** rejected.

### Anchor Phase 18 on swift-publicsuffix v0.2 (ICANN/PRIVATE split)

**Pros:** closes a quiet Phase 13 deferral.

**Cons:** narrow scope. Better as a v0.1.1 patch when an adopter needs it.

**Verdict:** rejected as Phase 18 anchor.

### Combine swift-distributed-tracing-bridge v0.2 with log + metrics bridge v0.2 bumps

**Pros:** "complete trinity v0.2" packaging.

**Cons:** the log + metrics bridges **auto-benefit** from the tracing bridge's W3C extract — they read `ServiceContext.otlpTraceIDs` which the bridge now populates from inbound headers. No code changes needed in the log + metrics bridges. Bundling them as v0.2 bumps would be churn without benefit.

**Verdict:** rejected. Single-tranche bridge-only is the right shape.

## Resolution

**Accepted 2026-05-14.** Phase 18 plans may begin authoring. Single existing package: swift-distributed-tracing-bridge v0.2.

Phase 18 closes the cross-process gap left after Phase 17's trinity completion. After Phase 18, an HTTP middleware using `InstrumentationSystem.instrument.extract / .inject` (with any standard swift-distributed-tracing-compatible Extractor / Injector) will automatically propagate W3C-spec `traceparent` headers across service boundaries — and the log + metrics bridges will auto-correlate using the cross-process trace IDs with no additional changes.

swift-jwt-verify v0.2 signing, swift-idna v0.3, swift-brotli v0.3 streaming, B3 + Jaeger propagation, and OAuth 2.0 remain on the queue for Phase 19+.

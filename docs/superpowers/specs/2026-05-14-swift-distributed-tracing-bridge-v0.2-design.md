# swift-distributed-tracing-bridge v0.2 (W3C TraceContext) Design

**Phase 18 Tranche 18A** (RFC-0023). Existing package; minor version bump.

**Date:** 2026-05-14

---

## Goal

Wire W3C TraceContext extract/inject into Apple `Instrument` protocol's no-op v0.1 stubs in `OTLPTracer`. Use swift-tracing-otlp v0.3's already-shipped `OTLP.TraceContext.parse(traceparent:)` + `.traceparent` accessor (Phase 14B). Closes the in-process-only limitation of the Apple-frontend → bare-swift-OTLP adapter trinity.

## Scope

**In scope:**
- Replace `OTLPTracer.extract(_:into:using:)` no-op body with W3C `traceparent` header reader.
- Replace `OTLPTracer.inject(_:into:using:)` no-op body with W3C `traceparent` header writer.
- Bump dep: swift-tracing-otlp 0.1.0 → 0.3.0 (brings `OTLP.TraceContext`).
- 10 new tests via a `[String: String]`-backed fake Extractor/Injector.

**Out of scope (deferred to v0.3+):**
- B3 propagation (`X-B3-TraceId` / `X-B3-SpanId`).
- Jaeger propagation (`uber-trace-id`).
- `tracestate` header extract/inject.
- HTTP middleware bindings (out of scope for the bridge package itself).

**Anti-goals:**
- No breaking changes to existing v0.1 API surface.
- No new dependencies beyond the swift-tracing-otlp version bump.
- No retroactive changes to swift-log-bridge / swift-metrics-bridge (they auto-benefit via cascading `ServiceContext.otlpTraceIDs`).

---

## File changes

```
swift-distributed-tracing-bridge/
  Package.swift                              ← bump swift-tracing-otlp dep
  Sources/DistributedTracingBridge/
    OTLPTracer.swift                         ← extract + inject bodies + import
  Tests/DistributedTracingBridgeTests/
    W3CTraceContextTests.swift               ← NEW (10 tests + fake carriers)
  CHANGELOG.md                               ← v0.2.0 entry
  README.md                                  ← install version bump + brief cross-process section
```

Net LOC: ~30 source + ~150 tests = ~180 LOC.

---

## Public API changes

**No new public types.** The two `Instrument` protocol method bodies change behavior, but signatures and external contract stay identical.

| Symbol | v0.1 behavior | v0.2 behavior |
|---|---|---|
| `OTLPTracer.extract(_:into:using:)` | no-op | reads `"traceparent"` header → populates `ServiceContext.otlpTraceIDs` |
| `OTLPTracer.inject(_:into:using:)` | no-op | reads `ServiceContext.otlpTraceIDs` → writes `"traceparent"` header |

**Non-breaking:** v0.1 callers that invoked extract/inject got nothing; v0.2 callers get the W3C behavior. Same method signatures.

---

## Implementation sketches

### Extract

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

- Reads `"traceparent"` from the Carrier via the Extractor.
- Returns silently if header is missing or malformed (strict W3C parsing already validated in swift-tracing-otlp's 13-test parse suite).
- On valid parse, populates `ServiceContext.otlpTraceIDs` from the returned `OTLP.TraceContext`.
- W3C flags byte is discarded by v0.2 (no per-call sampling decision); cascading reads via `OTLP.TraceContext.current(in:)` re-apply `0x01` default.

### Inject

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

- No active `ServiceContext.otlpTraceIDs` → no header written (silent no-op).
- Constructs `OTLP.TraceContext` with `traceFlags: 0x01` (sampled default).
- `OTLP.TraceContext.traceparent` returns `nil` if `traceID` or `spanID` are not the spec-required 16 / 8 bytes — silently no-op in that case (defensive; shouldn't happen with values originating from `OTLPTracer.startSpan`).
- Writes under key `"traceparent"` (lowercase canonical).

---

## Test plan (10 tests, 1 suite)

`Tests/DistributedTracingBridgeTests/W3CTraceContextTests.swift`:

Fake carrier:

```swift
private struct DictExtractor: Extractor {
    let dict: [String: String]
    func extract(key: String, from carrier: [String: String]) -> String? { carrier[key] }
}
private struct DictInjector: Injector {
    func inject(_ value: String, forKey key: String, into carrier: inout [String: String]) {
        carrier[key] = value
    }
}
```

Tests:

1. **Extract with canonical W3C traceparent** populates `ServiceContext.otlpTraceIDs` with the expected traceID + spanID.
2. **Extract with no traceparent header** leaves `ServiceContext.otlpTraceIDs` unchanged (nil if previously unset).
3. **Extract with wrong-length traceparent** (e.g., `"00-abc-def-01"`) is silently rejected.
4. **Extract with uppercase hex** (e.g., `"00-4BF92F35...-...-01"`) is silently rejected per W3C strict parsing.
5. **Extract with all-zero traceID** (`"00-00000000000000000000000000000000-..."`) is silently rejected.
6. **Extract with non-`00` version** (`"01-...-..."`) is silently rejected.
7. **Inject with no active otlpTraceIDs** writes nothing to the carrier dict.
8. **Inject with valid otlpTraceIDs** writes a well-formed traceparent header (validates length + dash positions).
9. **Round-trip: inject → extract preserves IDs.** Inject a context, extract from the resulting carrier, verify the extracted IDs match the original.
10. **Chain test: extract → startSpan propagates parent IDs.** Extract a parent context, start a child span via `OTLPTracer.startSpan(_:context:...)`, verify the child's `parentSpanID` matches the extracted spanID and the trace IDs match.

All tests Foundation-free; vectors as Swift literals matching swift-tracing-otlp's TraceContextTests.

---

## CHANGELOG entry

```markdown
## [0.2.0] - 2026-05-14

### Added
- W3C TraceContext extract/inject in `OTLPTracer.extract(_:into:using:)` and `OTLPTracer.inject(_:into:using:)`. v0.1's no-op stubs are replaced with real implementations using swift-tracing-otlp v0.3's `OTLP.TraceContext.parse(traceparent:)` + `.traceparent` accessor.
- 10 new tests covering W3C propagation: valid extract, missing header, malformed length, uppercase hex (rejected), all-zero IDs (rejected), non-00 version (rejected), inject with no context, valid inject shape, round-trip preservation, chain-with-startSpan.

### Changed
- swift-tracing-otlp dep bumped 0.1.0 → 0.3.0 (brings `OTLP.TraceContext` value type added in Phase 14B).

### Migration (v0.1 → v0.2)
- **Non-breaking.** v0.1 callers using `extract` / `inject` got no-ops; v0.2 callers get W3C behavior. Same method signatures.
- Adopters who previously plumbed `traceparent` manually in their HTTP middleware can now rely on `InstrumentationSystem.instrument.extract` / `.inject`.

### Cascading benefit
- swift-log-bridge v0.1 + swift-metrics-bridge v0.1 auto-pick-up cross-process trace IDs via `ServiceContext.otlpTraceIDs` (which v0.2 now populates from inbound HTTP `traceparent` headers). No code changes required in those packages.

### Phase 18
- Tranche 18A of [RFC-0023](https://github.com/bare-swift/bare-swift/blob/main/rfcs/0023-phase-18-anchor-swift-distributed-tracing-bridge-w3c.md). Closes the in-process-only limitation of the Apple-frontend → bare-swift-OTLP adapter trinity.
```

---

## README updates

- Bump install snippet `from: "0.1.0"` → `from: "0.2.0"`.
- Add a "Cross-process propagation" section after Usage:

```markdown
## Cross-process propagation

Since v0.2, the bridge implements W3C TraceContext (`traceparent` header)
extract + inject via the standard `Instrument` protocol methods. HTTP
middleware that uses `InstrumentationSystem.instrument.extract` and
`.inject` automatically propagates trace context across service boundaries.

```swift
// Inbound (server middleware): extract from HTTP headers into context.
var context = ServiceContext.topLevel
InstrumentationSystem.instrument.extract(headers, into: &context, using: HTTPExtractor())

// Outbound (HTTP client): inject context into outgoing headers.
var headers: [String: String] = [:]
InstrumentationSystem.instrument.inject(context, into: &headers, using: HTTPInjector())
```

`B3` and `Jaeger` header formats are deferred to v0.3+.
```

---

## CI considerations

No CI workflow changes. Existing reusable workflow handles the build/test/sanitizers/docc matrix. Sanitizers stay ON (no large static data in this package).

---

## Ship sequencing

1. Bump `swift-tracing-otlp` dep in `Package.swift` from 0.1.0 to 0.3.0.
2. Add `import TracingOTLP` to `Sources/DistributedTracingBridge/OTLPTracer.swift`.
3. Replace `extract` body.
4. Replace `inject` body.
5. Write `Tests/DistributedTracingBridgeTests/W3CTraceContextTests.swift` (10 tests + fake Extractor/Injector).
6. Run `swift test` to confirm all v0.1 + new v0.2 tests pass.
7. Update CHANGELOG with v0.2.0 entry.
8. Update README install version + "Cross-process propagation" section.
9. Commit, push, watch CI.
10. Tag v0.2.0; watch Release + Publish docs.
11. Umbrella bump (`swift-distributed-tracing-bridge` 0.1.0 → 0.2.0 in `packages/index.json`).
12. Memory closure (`project_phase18_18a_shipped.md`).

Total Phase 18 calendar: ~0.5 day wall-clock (matches RFC-0023 estimate).

---

## Decisions log (locked)

- Header key `"traceparent"` (lowercase canonical per W3C / RFC 7230 case-insensitive).
- Trace flags hardcoded `0x01` on inject (sampled default). Future v0.3 may surface a configurable sampler.
- Silent no-op on missing/malformed headers (defensive — propagation is best-effort).
- W3C strict parsing inherited from swift-tracing-otlp's `OTLP.TraceContext.parse(traceparent:)`.
- `OTLPTraceIDs` (swift-distributed-tracing-bridge v0.1's local type) bridges to `OTLP.TraceContext` (swift-tracing-otlp v0.3's value type) through traceID + spanID fields only. The W3C `traceFlags` byte is captured during parse but discarded after constructing OTLPTraceIDs; reads via `OTLP.TraceContext.current(in:)` (swift-log-bridge + swift-metrics-bridge) re-apply `0x01` default.
- Non-breaking; no migration needed.
- No new umbrella tier; no new package; existing `observability` tier entry just bumps version.

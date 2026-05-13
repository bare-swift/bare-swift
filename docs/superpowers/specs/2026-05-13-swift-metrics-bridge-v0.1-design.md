# swift-metrics-bridge v0.1 Design

**Phase 17 Tranche 17A** (RFC-0022). New package — 48th in the ecosystem.

**Date:** 2026-05-13

---

## Goal

Bridge Apple's `swift-metrics` `MetricsFactory` onto OTLP metric records buffered for caller-driven export. Closes the symmetric gap left after Phase 16: traces ✓ (swift-distributed-tracing-bridge), logs ✓ (swift-log-bridge), metrics now via this package. Cross-signal exemplars auto-attach via `ServiceContext.otlpTraceIDs`.

## Scope (Shape A — single tranche, full v0.1)

**In scope:**
- `OTLPMetricsFactory` conforming to Apple `MetricsFactory`.
- 6 internal handler classes: `OTLPCounter`, `OTLPFloatingPointCounter`, `OTLPHistogramRecorder`, `OTLPGaugeRecorder`, `OTLPTimer`, `OTLPMeter`.
- `flushExport() -> Bytes` (OTLP/HTTP/protobuf body) + `takeBufferedMetrics() -> [OTLP.Metric]`.
- Cross-signal exemplar auto-attach (most-recent-only).
- `Configuration` struct with separate `defaultHistogramBuckets` + `defaultTimerBuckets`.
- `MetricsBridgeError` typed-throws (empty for v0.1).

**Out of scope (deferred to v0.2+):**
- gRPC OTLP transport.
- HTTP client wiring (caller supplies).
- Delta aggregation temporality.
- Reservoir sampling for exemplars.
- Per-metric bucket override.
- Aggregation views.
- Periodic auto-flush.
- Resource attribute auto-detection.

**Anti-goals:**
- No Foundation in `Sources/MetricsBridge` or test target.
- No swift-atomics dep (Mutex-based, matches bridge family).
- No breaking changes to swift-otlp-exporter or swift-prometheus-metrics.

---

## Module structure

```
Sources/MetricsBridge/
  Documentation.docc/
    MetricsBridge.md
  MetricsBridge.swift              ← top-level doc enum (~15 LOC)
  OTLPMetricsFactory.swift         ← factory + registry + flushExport (~180 LOC)
  OTLPHandlers.swift               ← 6 handler classes + Snapshottable (~280 LOC)
  Configuration.swift              ← bucket defaults (~50 LOC)
  Dimensions.swift                 ← [(String,String)] → [OTLP.KeyValue] (~25 LOC)
  WallClock.swift                  ← clock_gettime via Darwin/Glibc (~30 LOC)
  MetricsBridgeError.swift         ← empty enum (~10 LOC)
Tests/MetricsBridgeTests/
  OTLPMetricsFactoryTests.swift    ← 10 tests
  OTLPCounterTests.swift           ← 5 tests
  OTLPFloatingPointCounterTests.swift  ← 3 tests
  OTLPRecorderTests.swift          ← 5 tests
  OTLPTimerTests.swift             ← 3 tests
  OTLPMeterTests.swift             ← 2 tests
  DimensionsMappingTests.swift     ← 2 tests
  ExemplarsTests.swift             ← 4 tests
  MetricsBridgeErrorTests.swift    ← 2 tests
```

Total source: ~590 LOC. Tests: ~600 LOC across 9 suites, 36 tests.

---

## Public API

```swift
public final class OTLPMetricsFactory: MetricsFactory, @unchecked Sendable {
    public init(
        resource: OTLP.Resource = OTLP.Resource(),
        scope: OTLP.InstrumentationScope = OTLP.InstrumentationScope(),
        configuration: Configuration = Configuration(),
        nextWallInstant: @escaping @Sendable () -> Time.Instant = defaultWallClock
    )

    // MetricsFactory conformance (5 make + 5 destroy).
    public func makeCounter(label: String, dimensions: [(String, String)]) -> CounterHandler
    public func makeFloatingPointCounter(label: String, dimensions: [(String, String)]) -> FloatingPointCounterHandler
    public func makeRecorder(label: String, dimensions: [(String, String)], aggregate: Bool) -> RecorderHandler
    public func makeMeter(label: String, dimensions: [(String, String)]) -> MeterHandler
    public func makeTimer(label: String, dimensions: [(String, String)]) -> TimerHandler
    public func destroyCounter(_ handler: CounterHandler)
    public func destroyFloatingPointCounter(_ handler: FloatingPointCounterHandler)
    public func destroyRecorder(_ handler: RecorderHandler)
    public func destroyMeter(_ handler: MeterHandler)
    public func destroyTimer(_ handler: TimerHandler)

    // Bridge-specific.
    public func flushExport() -> Bytes
    public func takeBufferedMetrics() -> [OTLP.Metric]
    public var registeredHandlerCount: Int { get }
}

public struct Configuration: Sendable {
    public var defaultHistogramBuckets: [Double]
    public var defaultTimerBuckets: [Double]

    public init(
        defaultHistogramBuckets: [Double] = Configuration.defaultHistogramBuckets,
        defaultTimerBuckets: [Double] = Configuration.defaultTimerBuckets
    )

    /// Prometheus-standard log-spaced buckets: 5ms → 10s.
    public static let defaultHistogramBuckets: [Double] = [
        0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10
    ]

    /// Timer buckets in seconds (Timer divides nanos by 1e9). Same shape.
    public static let defaultTimerBuckets: [Double] = [
        0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10
    ]
}

public enum MetricsBridgeError: Error, Equatable, Sendable {}

/// Default wall clock via POSIX clock_gettime(CLOCK_REALTIME).
public let defaultWallClock: @Sendable () -> Time.Instant
```

`Configuration` is a value type carried in the factory init; once installed via `MetricsSystem.bootstrap`, it's immutable.

---

## Internal types

### Snapshottable protocol

```swift
internal protocol Snapshottable: Sendable {
    func snapshot(at timeUnixNano: UInt64) -> OTLP.Metric
}
```

All 6 handler classes conform. Factory iterates via this protocol in `flushExport()` / `takeBufferedMetrics()`.

### Handler classes

All `final class @unchecked Sendable`. State protected by `Synchronization.Mutex`.

**OTLPCounter — CounterHandler**

```swift
internal struct State {
    var value: Int64
    var exemplar: OTLP.Exemplar?
}
private let state: Mutex<State>

func increment(by amount: Int64)        // ←  captures exemplar if active context
func reset()                            // sets state.value = 0
```

Snapshot: `OTLP.Metric.Data.sum(OTLP.Sum(dataPoints:[OTLP.NumberDataPoint(asInt: state.value, attributes: ..., exemplars: [state.exemplar].compactMap{ $0 })], aggregationTemporality: .cumulative, isMonotonic: true))`.

**OTLPFloatingPointCounter — FloatingPointCounterHandler**

Same shape, `Double` value, `.asDouble`, `isMonotonic: true`.

**OTLPHistogramRecorder — RecorderHandler (aggregate: true)**

```swift
internal struct State {
    var bucketCounts: [UInt64]   // cumulative counts per bucket boundary
    var sum: Double
    var count: UInt64
    var exemplar: OTLP.Exemplar?
}
```

`record(_ value: Int64)` and `record(_ value: Double)` both convert to Double, increment all buckets where `value ≤ boundary`, accumulate sum + count, capture exemplar.

Snapshot: `OTLP.Histogram` data point with explicit bucket boundaries from configuration.

**OTLPGaugeRecorder — RecorderHandler (aggregate: false)**

Most-recent value only. Snapshot: `OTLP.Gauge`.

**OTLPTimer — TimerHandler**

```swift
func recordNanoseconds(_ duration: Int64)
```

Internally divides by 1e9 → seconds, then records against `defaultTimerBuckets`. Snapshot: `OTLP.Histogram`.

**OTLPMeter — MeterHandler**

```swift
func set(_ value: Int64)
func set(_ value: Double)
func increment(by amount: Double)
func decrement(by amount: Double)
```

Snapshot: `OTLP.Gauge`.

### Registry

```swift
internal struct RegistryEntry {
    let handler: any Snapshottable
    let label: String
    let dimensions: [(String, String)]
}
private let registry: Mutex<[ObjectIdentifier: RegistryEntry]>
```

Keyed by handler `ObjectIdentifier` so `destroy*` can find + remove. Iteration order is non-deterministic — OTLP doesn't require ordering.

### Exemplar capture

Helper on each handler:

```swift
internal static func captureExemplar(
    value: Double,
    timeUnixNano: UInt64
) -> OTLP.Exemplar? {
    guard let ids = ServiceContext.current?.otlpTraceIDs else { return nil }
    return OTLP.Exemplar(
        filteredAttributes: [],
        timeUnixNano: timeUnixNano,
        value: .asDouble(value),
        spanID: ids.spanID,
        traceID: ids.traceID
    )
}
```

Stored under the handler's mutex on every measurement call. Most-recent wins.

### Dimensions mapping

```swift
internal func toKeyValues(_ dimensions: [(String, String)]) -> [OTLP.KeyValue] {
    dimensions.map { OTLP.KeyValue(key: $0.0, value: .string($0.1)) }
}
```

Trivial; lifted for testability.

---

## Aggregation temporality

Hardcoded `OTLP.AggregationTemporality.cumulative` for v0.1. swift-metrics-native semantics: counters report lifetime totals; histograms report cumulative bucket counts since handler creation. Delta deferred to v0.2.

---

## Bucket layout

Single set of bucket boundaries per category (histogram vs timer), set at factory init via `Configuration`. All histogram recorders share the same boundaries; all timers share the same boundaries. Per-metric override deferred.

Default boundaries match Prometheus / swift-prometheus-metrics: 5ms → 10s, log-spaced (11 buckets).

---

## CI considerations

Standard reusable workflow. `docc-target: MetricsBridge`. **Sanitizers ON** (no large static data). Strict-concurrency mandatory.

Pre-emptive Pages setup per Gate 16's codified three-API-call sequence (build_type=workflow → environment with custom_branch_policies=true → v* tag policy).

---

## Test plan (~36 tests, 9 suites)

### OTLPMetricsFactoryTests (10)
1. makeCounter returns a CounterHandler.
2. makeFloatingPointCounter returns a FloatingPointCounterHandler.
3. makeRecorder(aggregate:true) returns histogram-shaped handler.
4. makeRecorder(aggregate:false) returns gauge-shaped handler.
5. makeMeter returns a MeterHandler.
6. makeTimer returns a TimerHandler.
7. destroyCounter removes from registry.
8. registeredHandlerCount tracks state.
9. flushExport on empty factory returns empty Bytes.
10. flushExport drains the buffer; subsequent flush is empty.

### OTLPCounterTests (5)
1. increment by Int64 accumulates value.
2. snapshot emits `Sum` with `isMonotonic: true`, `cumulative`.
3. multiple increments accumulate.
4. concurrent increments are thread-safe (100 task group).
5. reset zeroes the value.

### OTLPFloatingPointCounterTests (3)
1. increment by Double accumulates.
2. snapshot emits `Sum` with `Double` data point.
3. multiple increments accumulate.

### OTLPRecorderTests (5)
1. aggregate=true: histogram bucket counts increment correctly per record.
2. aggregate=true: sum + count populated.
3. aggregate=false: snapshot emits `Gauge` with last value.
4. aggregate=true: custom buckets via Configuration honored.
5. aggregate=true: Int64 record converts to Double.

### OTLPTimerTests (3)
1. recordNanoseconds(N) records N/1e9 seconds against timer buckets.
2. snapshot emits `Histogram`.
3. Multiple records accumulate count/sum.

### OTLPMeterTests (2)
1. set + increment + decrement track state.
2. snapshot emits `Gauge`.

### DimensionsMappingTests (2)
1. `[("a","1"),("b","2")]` → 2 `OTLP.KeyValue` with `.string` values.
2. Empty dimensions → empty attributes.

### ExemplarsTests (4)
1. Counter inside ServiceContext with `otlpTraceIDs` → exemplar attached on snapshot.
2. Counter outside context → no exemplar (`nil`).
3. Histogram inside context → exemplar on histogram data point.
4. Subsequent measurement overwrites prior exemplar.

### MetricsBridgeErrorTests (2)
1. Conformances (Error, Equatable, Sendable).
2. Empty enum exhaustiveness placeholder.

All Foundation-free. Vectors as Swift literals.

---

## LOC budget

| Component | LOC |
|---|---|
| OTLPMetricsFactory.swift | ~180 |
| OTLPHandlers.swift | ~280 |
| Configuration.swift | ~50 |
| Dimensions.swift | ~25 |
| WallClock.swift | ~30 |
| MetricsBridge.swift + MetricsBridgeError.swift | ~25 |
| Tests (9 suites) | ~600 |
| **Total** | **~1190** |

Source ~590 LOC. Tests ~600 LOC. Comparable to swift-log-bridge (Phase 16A: ~245 + ~400).

---

## Migration

New package — no migration. Adopters using `PrometheusMetricsFactory` for metrics export today can switch to `OTLPMetricsFactory` by changing the bootstrap call:

```swift
// Before:
MetricsSystem.bootstrap(PrometheusMetricsFactory(...))

// After:
let otlpFactory = OTLPMetricsFactory(resource: ..., scope: ...)
MetricsSystem.bootstrap(otlpFactory)

// Drain periodically:
let payload: Bytes = otlpFactory.flushExport()
```

---

## Ship sequencing

1. `bare-swift new`-style scaffold (mkdir + Package.swift + LICENSE + NOTICE + README + CHANGELOG + MAINTAINERS + SECURITY + .gitignore + 3 workflows).
2. `gh repo create bare-swift/swift-metrics-bridge`.
3. **Pre-emptive Pages setup** (codified three-API-call sequence from Gate 16).
4. Implement source files (7 files, ~590 LOC).
5. Implement test suites (9 suites, ~36 tests).
6. CHANGELOG + README + DocC catalog.
7. Push, watch CI, fix if needed.
8. Tag v0.1.0; watch Release + Publish-docs.
9. Umbrella: add swift-metrics-bridge to `bare-swift/packages/index.json` (48th package).
10. Memory closure: `project_phase17_17a_shipped.md`.

Total Phase 17 calendar: ~1.5 days wall-clock (matches RFC-0022 Shape A estimate).

---

## Decisions log (locked)

- **Shape A** (single tranche, full v0.1 with exemplars).
- Package name `swift-metrics-bridge`; module name `MetricsBridge`. Matches established bridge naming convention.
- 6 handler classes mirror swift-prometheus-metrics structure.
- `Synchronization.Mutex` per handler (no swift-atomics dep).
- Cumulative aggregation temporality only.
- Most-recent-exemplar-only (no reservoir).
- Factory-level `Configuration` for histogram + timer buckets (no per-metric override).
- Snapshottable protocol for registry uniformity.
- WallClock duplicated from swift-log-bridge (matches established "each bridge gets its own minimal copy" pattern; ~30 LOC).
- Foundation-free public API; generation tools (if any) live outside `Sources/`.
- DocC catalog: standard topic structure with cross-package references in prose only (per [[feedback-docc-cross-package]]).

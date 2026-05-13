# swift-log-bridge v0.1 Design

**Phase 16 Tranche 16A** (RFC-0021, reshaped). New package — 47th in the ecosystem.

**Date:** 2026-05-13

---

## Goal

Add an `OTLPLogHandler` for Apple's `swift-log` that emits `OTLP.LogRecord` with automatic trace correlation via the active swift-distributed-tracing `ServiceContext`. Closes Phase 14's documented "implicit-context bridge" gap for the log side.

## Scope reshape from RFC-0021

RFC-0021 proposed `swift-distributed-tracing-otlp` containing `OTLPTracer` + `OTLPLogHandler` + `OTLP.TraceContext.current`. Brainstorm phase surfaced that `swift-distributed-tracing-bridge` v0.1 (Phase 3B) already provides `OTLPTracer` + `OTLPTraceIDs` + `ServiceContext.otlpTraceIDs`. The actual gap is just the log side.

**Reshape:**
- Renamed to `swift-log-bridge` (matches the established `swift-distributed-tracing-bridge` naming convention).
- Module name: `LogBridge`.
- Scope-cut to LogHandler + TraceContext.current bridge only.
- Estimated ~250 LOC source + ~25 tests (vs RFC-0021's ~500-800 LOC).

This is the **fourth time** in the project that brainstorm-phase reality-check surfaced scope-cut from an RFC's optimistic estimate (swift-brotli v0.2 encoder, swift-idna v0.1 Punycode-only, swift-otlp-json v0.1 hand-rolled, now swift-log-bridge). Will document in Gate 16 retro.

## Anti-goals (hardcoded for v0.1)

- **No new OTLPTracer.** Already exists in swift-distributed-tracing-bridge.
- **No HTTP transport.** Caller wires `flushExport()` bytes to their HTTP client.
- **No sampler.** v0.2+ if adopter demand surfaces.
- **No batch/buffer-bounded/retry.** Buffer grows until flushed; adopter responsibility.
- **No W3C traceparent extract/inject at log time.** Reads `ServiceContext.otlpTraceIDs` only. swift-distributed-tracing-bridge v0.2 will add W3C extract/inject; this handler picks it up automatically when bridge ships v0.2.
- **No Foundation.** Test target and `Sources/LogBridge` stay Foundation-free.

---

## Module structure

```
Sources/LogBridge/
  Documentation.docc/
    LogBridge.md
  LogBridge.swift              ← top-level doc enum (~15 LOC)
  OTLPLogHandler.swift         ← LogHandler conformer + buffer + severity mapping (~160 LOC)
  TraceContextBridge.swift     ← OTLP.TraceContext.current extension (~30 LOC)
  LogBridgeError.swift         ← typed-throws (~10 LOC)
  WallClock.swift              ← clock_gettime via Darwin/Glibc (~30 LOC)
Tests/LogBridgeTests/
  OTLPLogHandlerTests.swift    ← 12 tests
  TraceContextCurrentTests.swift  ← 5 tests
  EndToEndTests.swift          ← 6 tests
  LogBridgeErrorTests.swift    ← 2 tests (exhaustiveness)
```

---

## Public API

```swift
public final class OTLPLogHandler: LogHandler, @unchecked Sendable {
    public var logLevel: Logger.Level
    public var metadata: Logger.Metadata
    public subscript(metadataKey key: String) -> Logger.Metadata.Value? { get set }

    /// Initialize an OTLP log handler.
    public init(
        resource: OTLP.Resource = OTLP.Resource(),
        scope: OTLP.InstrumentationScope = OTLP.InstrumentationScope(),
        logLevel: Logger.Level = .info,
        nextWallInstant: @escaping @Sendable () -> Time.Instant = defaultWallClock
    )

    /// Apple swift-log LogHandler conformance.
    public func log(
        level: Logger.Level,
        message: Logger.Message,
        metadata explicitMetadata: Logger.Metadata?,
        source: String,
        file: String,
        function: String,
        line: UInt
    )

    /// Drain buffered records and encode to OTLP/HTTP/protobuf body.
    public func flushExport() -> Bytes

    /// Drain buffered records without encoding (for custom transport).
    public func takeBufferedLogRecords() -> [OTLP.LogRecord]

    /// Count of records currently buffered.
    public var bufferedRecordCount: Int { get }
}

extension OTLP.TraceContext {
    /// Build a TraceContext from the active swift-distributed-tracing
    /// ServiceContext's `otlpTraceIDs`. Trace flags default to 0x01 (sampled).
    /// Returns nil if no active otlpTraceIDs.
    public static func current(in context: ServiceContext = .topLevel) -> OTLP.TraceContext?
}

public enum LogBridgeError: Error, Equatable, Sendable {
    // No cases in v0.1; defensive extension point. Mirrors LogOTLPError.
}

/// Default wall-clock that calls clock_gettime(CLOCK_REALTIME).
public let defaultWallClock: @Sendable () -> Time.Instant = { ... }
```

---

## Internals

### OTLPLogHandler buffering

- `final class` (not value-type struct) wrapping `Mutex<[OTLP.LogRecord]>`. Apple's `LogHandler` protocol allows class conformers; swift-distributed-tracing-bridge's `OTLPTracer` uses the same shape.
- Buffer grows unbounded until `flushExport()` / `takeBufferedLogRecords()` drains it.
- `flushExport()` returns `Bytes` ready for `POST /v1/logs` with `Content-Type: application/x-protobuf`. Empty buffer returns `Bytes()`.

### Severity mapping

| swift-log `Logger.Level` | `OTLP.SeverityNumber` |
|---|---|
| `.trace` | `.trace` (1) |
| `.debug` | `.debug` (5) |
| `.info` | `.info` (9) |
| `.notice` | `.info2` (10) |
| `.warning` | `.warn` (13) |
| `.error` | `.error` (17) |
| `.critical` | `.fatal` (21) |

Also fill `OTLP.LogRecord.severityText` with the swift-log level name (`"info"`, `"warning"`, etc.).

### Metadata mapping

`Logger.Metadata` is `[String: Logger.Metadata.Value]`. Map to `[OTLP.KeyValue]`:

- `.string(s)` → `OTLP.AnyValue.string(s)`
- `.stringConvertible(c)` → `.string(String(describing: c))`
- `.dictionary(d)` → `.kvlist(d.map { OTLP.KeyValue(key: $0, value: ...) })`
- `.array(a)` → `.array(a.map { ... })`

Merge handler's `var metadata` with the per-log-call `explicitMetadata` (call-site wins on key collision).

### Source/file/function/line attributes

Per OpenTelemetry semantic conventions:
- `source` → `code.namespace` attribute
- `file` → `code.filepath` attribute
- `function` → `code.function` attribute
- `line` → `code.lineno` attribute (as `.int(_:)`)

### Trace correlation

Each `log(...)` call:
1. Reads `ServiceContext.current?.otlpTraceIDs` (TaskLocal from swift-distributed-tracing).
2. If present, builds `OTLP.TraceContext(traceID: ids.traceID, spanID: ids.spanID, traceFlags: 0x01)`.
3. Uses `OTLP.LogRecord(timeUnixNano:, severityNumber:, severityText:, body:, attributes:, traceContext:)` from swift-log-otlp v0.3.
4. If no active context, fall back to the canonical `OTLP.LogRecord(...)` init with empty traceID/spanID.

### Default wall clock

```swift
public let defaultWallClock: @Sendable () -> Time.Instant = {
    #if canImport(Darwin)
    var ts = timespec()
    clock_gettime(CLOCK_REALTIME, &ts)
    return Time.Instant(nanosecondsSinceEpoch: Int64(ts.tv_sec) * 1_000_000_000 + Int64(ts.tv_nsec))
    #elseif canImport(Glibc)
    var ts = timespec()
    clock_gettime(CLOCK_REALTIME, &ts)
    return Time.Instant(nanosecondsSinceEpoch: Int64(ts.tv_sec) * 1_000_000_000 + Int64(ts.tv_nsec))
    #else
    return Time.Instant(nanosecondsSinceEpoch: 0)
    #endif
}
```

Precedent: swift-publicsuffix has `#if canImport(Darwin)` / `#elseif canImport(Glibc)` blocks for platform-conditional code.

### TraceContextBridge extension

`OTLP.TraceContext.current(in:)` reads the optional `ServiceContext.otlpTraceIDs`, returns `nil` if not present, otherwise constructs a TraceContext. The `in context: ServiceContext = .topLevel` default lets callers explicitly pass a context (typically `ServiceContext.current` for the active TaskLocal).

---

## Dependencies (6 packages)

**Apple:**
- `swift-log` 1.5.0+ (LogHandler protocol)
- `swift-distributed-tracing` 1.0.0+ (ServiceContext)

**bare-swift:**
- `swift-bytes` 0.1.0 (output)
- `swift-tracing-otlp` 0.3.0 (`OTLP.TraceContext`)
- `swift-log-otlp` 0.3.0 (`OTLP.LogRecord` + traceContext convenience init + `OTLP.encodeLogs(_:)`)
- `swift-distributed-tracing-bridge` 0.1.0 (`OTLPTraceIDs` + `ServiceContext.otlpTraceIDs`)
- `swift-time` 0.1.0 (`Time.Instant`)

Total transitive: swift-bytes, swift-varint, swift-otlp-exporter, swift-tracing-otlp, swift-log-otlp, swift-distributed-tracing-bridge, swift-time + Apple's swift-log + swift-distributed-tracing.

Deepest direct dep graph in the bare-swift ecosystem to date. Matches the integration-package shape and is intentional.

---

## Test plan (~25 tests)

### OTLPLogHandlerTests (12 tests)
1. Empty buffer flushes to empty `Bytes`.
2. `log(.info, ...)` adds a record with the message as body.
3. `log(.debug, ...)` filtered when `logLevel == .info`.
4. Handler's `var metadata` is merged into emitted records.
5. Call-site `explicitMetadata` overrides handler metadata on key collision.
6. `source`, `file`, `function`, `line` map to `code.*` attributes.
7. `takeBufferedLogRecords` drains the buffer.
8. `flushExport()` produces decodable OTLP bytes (round-trip via OTLP.encodeLogs).
9. All 7 severity mappings produce correct OTLP.SeverityNumber.
10. `nextWallInstant` injection: custom clock → expected `timeUnixNano`.
11. `bufferedRecordCount` accessor tracks state.
12. Multi-thread safety via Mutex (sanity check; multiple concurrent log calls).

### TraceContextCurrentTests (5 tests)
1. `OTLP.TraceContext.current(in: .topLevel)` returns nil.
2. `OTLP.TraceContext.current(in: ctx)` with `ctx.otlpTraceIDs` set returns matching TraceContext.
3. Returned trace flags default to 0x01 (sampled).
4. Returned traceID and spanID match the OTLPTraceIDs.
5. Default argument `in: .topLevel` is well-formed.

### EndToEndTests (6 tests)
1. `LoggingSystem.bootstrap` + log call + `flushExport()` round-trip.
2. Two records batched into one export request.
3. Per-`Logger` metadata copies don't share state across loggers.
4. Cross-signal: synthesize a `ServiceContext` with `otlpTraceIDs` set, log within it via `ServiceContext.withValue` or by passing context explicitly; emitted record's `traceID`/`spanID` match.
5. `flushExport()` followed by another log → second flush contains only the new record.
6. Resource + Scope from init propagate into the exported request structure.

### LogBridgeErrorTests (2 tests)
1. Empty enum exhaustiveness (matches LogOTLPError pattern; placeholder for v0.2 cases).
2. `LogBridgeError: Sendable, Equatable, Error` conformances.

All tests Foundation-free; vectors as Swift literals.

---

## CI considerations

Standard reusable workflow with `docc-target: LogBridge`. **Sanitizers ON** (no large static data). Strict-concurrency mandatory.

Initial repo setup requires the [[feedback-github-pages-new-repo]] pattern: enable Pages + v* tag deploy policy BEFORE the first tag.

---

## Migration notes

This is a v0.1 release of a new package — no migration. Adopters who currently maintain custom swift-log handlers for OTLP can drop them and use `OTLPLogHandler` directly.

---

## Ship sequencing

1. `bare-swift new swift-log-bridge` to scaffold (Pages + tag policy enabled pre-emptively).
2. Implement source files (5 files, ~250 LOC).
3. Implement test suites (4 suites, ~25 tests, ~400 LOC).
4. CHANGELOG + README + DocC.
5. CI: push, watch, fix if needed.
6. Tag v0.1.0; watch Release + Publish-docs.
7. Umbrella: add swift-log-bridge to `bare-swift/packages/index.json` (47th package, tier: `observability`).
8. Memory closure: `project_phase16_16a_shipped.md`.

Total Phase 16 calendar: ~1 day wall-clock.

---

## Decisions log (locked)

- **Rename + scope-cut from RFC-0021.** Package is `swift-log-bridge`, not `swift-distributed-tracing-otlp`. Scope is LogHandler + TraceContext.current bridge only (no OTLPTracer — already in swift-distributed-tracing-bridge).
- **Final class for LogHandler** (matches OTLPTracer pattern).
- **Buffer-and-flush model** (matches OTLPTracer's `flushExport()`; do not auto-export per log call).
- **Severity mapping** as specified above; explicit table avoids edge-case ambiguity.
- **Metadata recursive flattening** (`.dictionary` → `.kvlist`; `.array` → `.array`; `.stringConvertible` → `.string` via `String(describing:)`).
- **Trace correlation only from `ServiceContext.otlpTraceIDs`** (not W3C `traceparent` extract — defers to swift-distributed-tracing-bridge v0.2 when it adds W3C propagation).
- **Trace flags default 0x01 (sampled)** when correlation is active.
- **Default wall clock via `clock_gettime(CLOCK_REALTIME)`** with platform-conditional imports.
- **`nextWallInstant` injection point** for tests (deterministic timestamps).
- **No new umbrella tier** — fits `observability`.

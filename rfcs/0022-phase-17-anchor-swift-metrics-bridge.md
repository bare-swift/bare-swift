# RFC-0022 — Phase 17 anchor: swift-metrics-bridge

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-13 |
| Resolution | Accepted 2026-05-13 — Phase 17 plans may begin authoring. Single new package: swift-metrics-bridge. |

## Summary

Anchor Phase 17 on **swift-metrics-bridge** — a new adapter package that bridges Apple's [swift-metrics](https://github.com/apple/swift-metrics) into the bare-swift OTLP exporter family. Closes the symmetric gap left after Phase 16 (logs bridge done) and completes the Apple-frontend → bare-swift-OTLP adapter trinity:

- `swift-distributed-tracing-bridge` (traces) — Phase 3B v0.1
- `swift-log-bridge` (logs) — Phase 16A v0.1
- **`swift-metrics-bridge` (metrics)** — Phase 17A v0.1 (this RFC)

Single-anchor phase. Structurally similar to Phase 16A (swift-log-bridge): factory class with caller-driven flush, auto cross-signal exemplar attachment via `ServiceContext.otlpTraceIDs`.

## Problem

[Gate 16 retrospective](../docs/gates/2026-05-13-gate-16-retrospective.md) item 5 requires "Phase 17 anchor decision recorded as an RFC" before Phase 17 plans begin. The retrospective surveyed eight candidate waves; this RFC formalizes the choice.

**Concrete gap:** the bare-swift ecosystem has exactly one swift-metrics adapter today — `swift-prometheus-metrics` (Phase 2B), which routes Apple swift-metrics to Prometheus text exposition. **There is no OTLP equivalent.** Adopters who want to:

- Export traces via OTLP (`swift-tracing-otlp` + `swift-distributed-tracing-bridge`) ✓
- Export logs via OTLP (`swift-log-otlp` + `swift-log-bridge`) ✓
- Export metrics via OTLP

…have to manually build `OTLP.MetricData` records from their application code. The swift-metrics → OTLP path doesn't exist.

This asymmetry is conspicuous after Phase 16. Production observability stacks increasingly want OTLP-everywhere (traces + logs + metrics all using the same protocol, encoded for the same collector). Phase 17 closes the gap.

Additionally, **cross-signal exemplar correlation** — attaching trace IDs to metric data points so a percentile spike in a histogram can link back to the specific spans that contributed to it — is a Phase 14-era capability that's been waiting for the metrics bridge to land.

## Proposal

### Anchor: swift-metrics-bridge (single new package)

**swift-metrics-bridge v0.1** — adapter wiring Apple's swift-metrics into bare-swift's OTLP exporter family. Module name: `MetricsBridge`. Repo: `bare-swift/swift-metrics-bridge`.

| API | Purpose |
|---|---|
| `OTLPMetricsFactory` | Apple `MetricsFactory` protocol implementation. Per-metric handlers (`OTLPCounter`, `OTLPRecorder`, `OTLPTimer`) accumulate measurements. |
| `OTLPMetricsFactory.flushExport()` | Drains buffered metric snapshots and encodes as `OTLP.ExportMetricsServiceRequest` body bytes. |
| `OTLPMetricsFactory.takeBufferedMetrics()` | Drains without encoding (custom transport). |
| Auto exemplar attachment | When `ServiceContext.current?.otlpTraceIDs` is active during a measurement, attach an `OTLP.Exemplar` carrying the trace+span IDs to the recorded data point. |
| `MetricsBridgeError` | Typed-throws enum for failure modes during snapshotting. |

### Tranches

Single-anchor phase shape, two possible decompositions:

**Shape A (recommended): single tranche, full v0.1**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 17A | swift-metrics-bridge v0.1 | `OTLPMetricsFactory` + handlers + `flushExport()` + auto-exemplar attachment | ~1.5 days |

**Shape B (alternate): two tranches if exemplar wiring expands**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 17A | swift-metrics-bridge v0.1 | `OTLPMetricsFactory` + handlers + `flushExport()` (no exemplars) | ~1 day |
| 17B | swift-metrics-bridge v0.2 | Cross-signal exemplar auto-attach | ~0.5 day |

Decision deferred to brainstorm phase (verify the swift-metrics `MetricsFactory` protocol shape and Apple's per-handler-type semantics against `swift-prometheus-metrics` as the in-house reference implementation).

Per the now-canonical brainstorm-phase reality-check pattern (5th instance expected), Shape B is likely if exemplar wiring is wider than expected — the cross-signal correlation pattern from Phase 14/16 carries through cleanly but the per-data-point exemplar attachment has more spec surface than a single Logger metadata extraction.

### Tranche 17A scope sketch

`swift-metrics-bridge` v0.1 covers:

- **`OTLPMetricsFactory`** conforming to Apple `MetricsFactory`. Constructs per-handler classes on `makeCounter` / `makeRecorder` / `makeTimer` / `makeFloatingPointCounter` / `makeMeter`. Each handler is a `final class` registered with the factory's internal registry (keyed by `(label, dimensions)`).
- **`OTLPCounter`** conforming to `CounterHandler`. Per-handler atomic counter via `ManagedAtomic<Int64>` (matches swift-prometheus-metrics's pattern). Snapshotted as `OTLP.NumberDataPoint` with `Sum` aggregation.
- **`OTLPFloatingPointCounter`** conforming to `FloatingPointCounterHandler`. `ManagedAtomic<UInt64>` over `Double` bit pattern. Same snapshot path.
- **`OTLPRecorder`** conforming to `RecorderHandler`. Two flavors: `aggregate: true` (histogram — bucketed counts via `OTLP.HistogramDataPoint`) and `aggregate: false` (gauge — last value via `OTLP.NumberDataPoint`).
- **`OTLPTimer`** conforming to `TimerHandler`. Records as nanoseconds in an `OTLP.HistogramDataPoint`.
- **`OTLPMeter`** conforming to `MeterHandler`. Gauge handler.
- **`flushExport()`** — snapshots all handlers, builds an `OTLP.ExportMetricsServiceRequest`, encodes via `OTLP.encodeMetrics(_:)` from swift-otlp-exporter, returns `Bytes`.
- **`takeBufferedMetrics()`** — returns `[OTLP.Metric]` for custom transport.
- **Auto exemplar attachment:** every measurement call (`Counter.increment`, `Recorder.record`, `Timer.recordNanoseconds`) reads `ServiceContext.current?.otlpTraceIDs` and, if present, attaches an `OTLP.Exemplar(traceID:spanID:value:timeUnixNano:)` to the data point. Per OTLP spec, exemplars are a bounded ring per data point; v0.1 keeps the most recent N (configurable, default 1).
- **`MetricsBridgeError`** typed-throws enum (no cases in v0.1; forward-compat).

**Reuses existing types** (from swift-otlp-exporter):
- `OTLP.Metric`, `OTLP.Sum`, `OTLP.Gauge`, `OTLP.Histogram`, `OTLP.NumberDataPoint`, `OTLP.HistogramDataPoint`, `OTLP.Exemplar`
- `OTLP.AggregationTemporality.cumulative` (default; swift-metrics doesn't surface delta)
- `OTLP.Resource`, `OTLP.InstrumentationScope`, `OTLP.KeyValue`

**Out of scope for v0.1:**
- **HTTP transport.** Encoder-only; caller wires `Bytes` to their HTTP client.
- **gRPC OTLP transport.** Separate spec.
- **Delta aggregation temporality.** v0.1 emits cumulative only. swift-metrics surfaces lifetime totals natively.
- **Aggregation views / configurable buckets per histogram.** v0.1 uses a sensible default bucket layout (10-step log-scale 1..1e9 nanoseconds, matching swift-prometheus-metrics's default).
- **Resource attribute auto-detection (hostname, pid, etc.).** Caller supplies the `OTLP.Resource`.
- **Periodic auto-flush.** Caller drives `flushExport()` (matches OTLPTracer + OTLPLogHandler).
- **Reservoir sampling for exemplars.** v0.1 keeps the most recent N per data point (default 1).

### Why this anchor

1. **Closes the symmetric Phase 16 gap.** Logs bridge done, traces bridge done; metrics is the missing third.
2. **Concrete gap, not a design observation.** Adopters who want OTLP metrics via swift-metrics today have no path. Multiple production stacks in the OTLP migration cycle hit this.
3. **Audience continuity.** Same observability community that uses the other two bridges.
4. **Net-new package keeps SemVer churn manageable.** Phase 15 had two BREAKING bumps; Phase 16 was net-new (no breaking); Phase 17 stays on net-new.
5. **Risk profile is well-bounded.**
   - swift-metrics `MetricsFactory` protocol is stable (Apple's package, v2.x).
   - All `OTLP.*` metric types already exist in swift-otlp-exporter (Phase 2).
   - swift-prometheus-metrics is the in-house reference for the handler-class pattern.
   - Cross-signal exemplar wiring reuses the `OTLP.TraceContext.current(in:)` pattern from swift-log-bridge.
6. **Sets up Phase 18+ options.**
   - **swift-jwt-verify v0.2 signing** has had a full week to settle.
   - **swift-distributed-tracing-bridge v0.2** (W3C extract/inject) becomes the natural follow-on now that the trinity is feature-complete.
   - **swift-idna v0.3 / swift-brotli v0.3 / OAuth 2.0** stay available.

### Out of scope for Phase 17

- **swift-jwt-verify v0.2 signing.** Phase 18+ candidate.
- **swift-idna v0.3 (Bidi + ContextJ).** Phase 18+ candidate.
- **swift-brotli v0.3 streaming encoder.** Phase 18+ candidate.
- **swift-distributed-tracing-bridge v0.2 (W3C extract/inject).** Phase 18+ candidate.
- **OAuth 2.0 / crypto-adjacent.** Same standing rejections.
- **swift-publicsuffix v0.2.** Quiet patch when an adopter asks.
- **gRPC OTLP transport** in any package. Separate spec.

### Tooling expectations

Phase 17 reuses all Phase 7–16 tooling without extension:

- `bare-swift new` scaffold pattern (matching swift-log-bridge's directory layout).
- **Pre-emptive Pages setup** per Gate 16's codified three-API-call sequence:
  1. `gh api -X POST /repos/bare-swift/swift-metrics-bridge/pages -f 'build_type=workflow'`
  2. `gh api -X PUT /repos/bare-swift/swift-metrics-bridge/environments/github-pages` with `custom_branch_policies: true`
  3. `gh api -X POST /repos/bare-swift/swift-metrics-bridge/environments/github-pages/deployment-branch-policies -f 'name=v*' -f 'type=tag'`
- Strict-concurrency in CI stays mandatory.
- `docc-target: MetricsBridge` opt-in from scaffold time.
- Sanitizers ON (no large static data).
- Cross-library validation at ship time: round-trip `OTLPMetricsFactory` snapshots → bytes → decode via OTel reference collector locally.

### Versioning

| Package | New | Future |
|---|---|---|
| swift-metrics-bridge | v0.1 (new — MetricsFactory + handlers + exemplars) | v0.2 = delta aggregation temporality if adopter demand; v0.3 = aggregation views / configurable buckets; v0.4 = gRPC if Phase 19+ adds it |

Phase 17 introduces **1 new package**, taking the ecosystem from **47 to 48 packages**.

### Anti-goals

- **No gRPC.** Out of scope; separate OTLP spec.
- **No HTTP client.** Pure adapter.
- **No retroactive changes** to swift-otlp-exporter / swift-prometheus-metrics. Phase 17 is purely additive on top.
- **No delta temporality.** v0.1 emits cumulative only.
- **No reservoir sampling for exemplars.** Most-recent-N (default 1) only.
- **No new umbrella tier.** Fits existing `observability` tier.

## Alternatives considered

### Anchor Phase 17 on swift-jwt-verify v0.2 (signing + RS256/ES384/EdDSA)

**Pros:** closes the verify-only design boundary.

**Cons:** v0.1 (Phase 11) has had ~2 days of adopter signal time. Multi-algorithm scope is wider than a focused single-anchor phase wants. Phase 18+ when v0.1 has had a week.

**Verdict:** rejected as Phase 17.

### Anchor Phase 17 on swift-idna v0.3 (Bidi + ContextJ + ContextO)

**Pros:** closes Phase 15's documented deferral.

**Cons:** third v0.x bump on same package in three phases would be aggressive churn. Bidi+ContextJ rarely-hit. Phase 18+ when demand surfaces.

**Verdict:** rejected as Phase 17.

### Anchor Phase 17 on swift-brotli v0.3 (streaming encoder)

**Pros:** closes Phase 12 v0.2 deferral.

**Cons:** narrower audience than the metrics trinity-closure. Phase 18+ candidate.

**Verdict:** rejected as Phase 17.

### Anchor Phase 17 on swift-distributed-tracing-bridge v0.2 (W3C extract/inject)

**Pros:** would enable W3C-spec-compliant trace propagation across HTTP boundaries; swift-log-bridge would auto-benefit.

**Cons:** no cascading dependents asking for it yet. The trinity-closure (Phase 17 metrics) is higher-value first. Phase 18+ candidate.

**Verdict:** rejected as Phase 17.

### Anchor Phase 17 on OAuth 2.0 client primitives

**Pros:** composes naturally.

**Cons:** still no concrete adopter demand. OAuth 2.0 has many flows; picking the right v0.1 subset needs adopter input.

**Verdict:** rejected as Phase 17.

### Skip ahead, do crypto-adjacent (swift-blake3 / swift-siphash / swift-hkdf)

**Pros:** would complement swift-jwt-verify if signing ever lands.

**Cons:** swift-crypto remains entrenched. **13th consecutive rejection.**

**Verdict:** rejected.

### Anchor Phase 17 on swift-publicsuffix v0.2

**Pros:** closes a quiet Phase 13 deferral.

**Cons:** narrow scope. Better as a v0.1.1 patch.

**Verdict:** rejected.

## Resolution

**Accepted 2026-05-13.** Phase 17 plans may begin authoring. Single new package: swift-metrics-bridge.

Phase 17 completes the Apple-frontend → bare-swift-OTLP adapter trinity. After Phase 17, an adopter can bootstrap once and get OTLP-encoded traces + logs + metrics from standard Apple swift-distributed-tracing + swift-log + swift-metrics APIs, with cross-signal correlation (trace IDs auto-attached to log records AND metric exemplars).

swift-jwt-verify v0.2 signing, swift-distributed-tracing-bridge v0.2 (W3C extract/inject), swift-idna v0.3, swift-brotli v0.3 streaming, and OAuth 2.0 remain on the queue for Phase 18+.

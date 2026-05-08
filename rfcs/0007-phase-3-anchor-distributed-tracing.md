# RFC-0007 — Phase 3 anchor: distributed tracing wave

| Field | Value |
| --- | --- |
| Status | Draft |
| Author | bare-swift project lead |
| Created | 2026-05-08 |
| Resolution | _(filled at merge)_ |

## Summary

Anchor Phase 3 on the **distributed tracing wave**: ship swift-tracing-otlp (OTLP exporter for traces, reusing swift-otlp-exporter's ProtoWriter), swift-distributed-tracing-bridge (Apple's swift-distributed-tracing frontend → bare-swift backend, mirrors swift-prometheus-metrics shape), and one logs-or-baggage stretch deliverable. This completes the OpenTelemetry trio (metrics ✓ from Phase 2, traces from Phase 3, logs deferrable) and fully closes the observability tier.

The wave reuses ~80% of Phase 2's machinery — protobuf encoding via swift-bytes/swift-varint, the adapter pattern from swift-prometheus-metrics, the same plan/execute cadence — and targets the same audience that adopted Phase 2's exporters.

## Problem

[Gate 2 retrospective](../docs/gates/2026-05-08-gate-2-retrospective.md) item 5 requires "Phase 3 anchor decision recorded as an RFC" before Phase 3 plans begin. The retrospective surveyed five candidate waves; this RFC formalizes the choice.

Phase 2 shipped the metrics half of an OpenTelemetry-aligned observability tier (statsd, prometheus, OTLP/metrics, prometheus-metrics adapter). Half a tier ships nobody. Server stacks treat metrics + traces + logs as a single observability story; without traces, the bare-swift observability story is incomplete. Phase 3 finishes it.

The question is: do we finish observability now, or pivot to a different anchor (config/serialization, networking primitives, crypto)? The Gate 2 retrospective laid out the trade-offs; this RFC commits.

## Proposal

### Anchor: distributed tracing wave

Phase 3 primary work targets distributed tracing on top of the OpenTelemetry data model. Specifically:

- **swift-tracing-otlp** — OTLP/HTTP+protobuf exporter for **traces** (`opentelemetry.proto.trace.v1`). Reuses the `ProtoWriter` and proto3-omission helpers from swift-otlp-exporter (likely as a sibling-package or shared internal package). Output is `Bytes` ready for `HTTP POST /v1/traces`.
- **swift-distributed-tracing-bridge** — adapter from Apple's [`swift-distributed-tracing`](https://github.com/apple/swift-distributed-tracing) frontend (the `Tracer` / `InstrumentProtocol` types) to a bare-swift backend that buffers spans and hands them to swift-tracing-otlp. Same shape as swift-prometheus-metrics.
- **(Stretch) swift-log-otlp** — OTLP/HTTP exporter for **logs** (`opentelemetry.proto.logs.v1`). If Tranche 3A and 3B finish with budget remaining, this completes the OpenTelemetry trio. If not, defer to Phase 4.

### Tranches

| Tranche | Packages | Estimated duration |
|---|---|---|
| 3A | swift-tracing-otlp (proto-encoded traces; reuses swift-otlp-exporter's ProtoWriter) | ~1 day at observed pace; ~1–2 months at original budget |
| 3B | swift-distributed-tracing-bridge (Apple frontend → bare-swift backend) | ~1 day; ~1–2 months at original budget |
| 3C | swift-log-otlp (OTLP/HTTP exporter for logs) — STRETCH | ~1 day; ~1 month at original budget |

Total Phase 3 budget: 3–6 months calendar at original roadmap pace, or roughly a week at observed Phase 2 execution pace.

### Why this anchor

1. **Adoption continuity.** swift-otlp-exporter's audience (services that already settled on OpenTelemetry) needs traces next. Without traces, swift-otlp-exporter is half a story; with them, it's a complete OTLP backend.
2. **Risk profile is known.** swift-otlp-exporter validated the protobuf encoding approach end-to-end. swift-tracing-otlp uses the same `ProtoWriter`, the same proto3 default-omission rules, the same hand-rolled-type-1:1-mirrors-the-proto pattern. The hardest part (getting wire-format correct under TDD) is solved.
3. **Adapter pattern proven.** swift-prometheus-metrics demonstrated the swift-metrics → swift-prometheus shape; swift-distributed-tracing-bridge is the same shape with different protocols. The dimension-stability / first-wins / sanitization / broken-handler patterns carry over.
4. **swift-bytes / swift-varint already ship.** No new foundation work required; Phase 2's foundation pays off again.
5. **Zero new dependencies on the critical path.** swift-distributed-tracing (Apple) is the analog of swift-metrics (Apple) for Phase 2. Same transitively-Foundation-free guarantee, same RFC-0001 compliance posture.

### Tranche 3A scope sketch (subject to per-package brainstorm)

`swift-tracing-otlp` covers the OTLP `traces.v1` schema:

- `ExportTraceServiceRequest` (top-level for HTTP POST)
- `ResourceSpans → ScopeSpans → Span`
- `Span` (one of the largest OTLP messages: name, span_id, trace_id, parent_span_id, kind, start/end times, attributes, events, links, status)
- `Status` (code + message)
- `Span.Event` (timestamp + name + attributes)
- `Span.Link` (linked trace+span IDs + attributes)
- `SpanKind` enum
- Reuses the `Resource`, `InstrumentationScope`, `KeyValue`, `AnyValue` types — these should be extracted from swift-otlp-exporter into a shared internal target, OR swift-tracing-otlp depends on swift-otlp-exporter and re-uses them.

Decision deferred to per-package brainstorm: shared internal package vs. cross-package re-use. Both are workable.

### Tranche 3B scope sketch

`swift-distributed-tracing-bridge` implements Apple's `InstrumentProtocol` and `Tracer` protocols:

- `InstrumentationSystem.bootstrap(_:)` — analog of `MetricsSystem.bootstrap`.
- Span lifecycle: `tracer.startSpan(_:context:ofKind:at:function:file:line:)` returns a `Span`; `Span.end()` triggers buffering / flushing.
- Baggage propagation via `Instrument.extract` / `Instrument.inject` (W3C TraceContext at minimum; B3 stretch).
- Background flushing: probably one of (a) caller-driven `flush()` API, (b) every-N-spans threshold, or (c) periodic timer. Decision deferred to brainstorm.
- Output is `Bytes` (via swift-tracing-otlp) ready for HTTP POST. No HTTP transport in v0.1.

### Out of scope for Phase 3

- gRPC OTLP transport (HTTP+protobuf only, matching Phase 2's choice).
- JSON OTLP variant.
- W3C TraceContext propagation may ship in v0.1 of the bridge; B3 / Jaeger formats deferred.
- HTTP transport itself — caller wires URLSession / async-http-client / NIO.
- Sampling logic — bare-swift exporter forwards all spans; head/tail sampling is application policy.
- Continuous-profiling exporters (eBPF, etc.).

## Alternatives considered

### Anchor on the config & serialization wave (TOML, YAML, MessagePack, CBOR, JSONPath)

Rejected because:
- Each package is a full-spec effort (TOML 1.0, YAML 1.2 are weeks each on their own).
- Adoption-fit is "broad audience but each package competes with established alternatives" (per the Gate 2 retro). Nothing in the wave has the same pull as "complete OTLP coverage".
- Better fit for Phase 4 once tracing closes the observability story.

### Anchor on networking primitives (swift-uri, swift-http-types-extras, swift-cookie, swift-mime)

Rejected — though strong candidate — because:
- Phase 2's observability anchor leaves a half-finished story; finishing it has higher leverage than starting a new tier.
- Networking primitives are individually small but the WHATWG URL parser alone is a multi-week effort.
- Better fit for Phase 4 alongside config/serialization.

### Anchor on crypto-adjacent (swift-blake3, swift-siphash, swift-hkdf, swift-x25519-bridge)

Rejected because:
- swift-crypto is the established Apple-backed option; differentiation requires more thought than Phase 3 has budget for.
- Adoption-fit is medium at best; users with crypto needs already have answers.

### Anchor on logging only (no tracing)

Rejected because:
- swift-log already exists with mature adapters; the only differentiation we'd add is OTLP-log-envelope output, which is one package not three.
- Logging without tracing leaves the OTLP story two-thirds done, not whole.

### Defer Phase 3 entirely; spend the time on Phase 1/2 polish

Rejected because:
- Phase 1/2 packages are stable (CI green, test coverage comprehensive, no open bugs visible).
- Polish work is endless; without an anchored next-wave, the project loses momentum.
- The Gate 2 retrospective's stop conditions did not trigger (calendar time well within budget; no security incidents; no Apple announcements).

## Drawbacks

1. **Continued specialization.** Phase 3 deepens the observability tier rather than broadening to config/networking/crypto. A project that thought it was getting a general-purpose Swift ecosystem might experience this as narrowness. Mitigation: state explicitly in the launch communication that Phase 3 is the natural completion of Phase 2 and Phase 4 will broaden.
2. **Apple dependency.** swift-distributed-tracing-bridge depends on Apple's swift-distributed-tracing (analog to Phase 2's swift-metrics dep). This is the second non-bare-swift dependency in the ecosystem. RFC-0001 is preserved (swift-distributed-tracing has minimal transitive deps, no Foundation in public API), but the precedent expands. RFC-0009 (inter-package dependency policy, candidate from Gate 2) becomes more pressing.
3. **Trace data is large.** OTLP trace payloads are bigger than metrics payloads (per-span attributes, events, links). The Bytes-allocation profile under load matters; we may discover swift-bytes optimizations (e.g., pooled buffers) are needed. v0.1 ships without optimization; document the trade-off.
4. **swift-distributed-tracing has had API churn.** The protocol set is more complex than swift-metrics's, with `Instrument`, `Tracer`, `Span`, `Baggage`, `ServiceContext`. We may target a specific minor version and document the constraint.
5. **Sampling is a real design problem.** v0.1 of swift-distributed-tracing-bridge will not sample; it will buffer all spans. For high-volume services this is unrealistic. v0.2 needs to add at least head sampling. Out of scope for the anchor; decision deferred to per-package brainstorm.

## Unresolved questions

These are intentionally deferred to per-package brainstorm sessions, not to the anchor RFC:

- **Shared internal target vs. cross-package re-use** for OTLP common types (`Resource`, `InstrumentationScope`, `KeyValue`, `AnyValue`, `ProtoWriter`, etc.). Either swift-tracing-otlp depends on swift-otlp-exporter and re-uses them, or we extract a `swift-otlp-common` package, or we duplicate. Trade-offs map cleanly.
- **Span buffering strategy** (caller-driven flush, threshold-driven, periodic). All three are valid; decision is driven by what real services need.
- **Sampling policy** for v0.2 (none in v0.1).
- **Baggage propagation formats** beyond W3C TraceContext (B3, Jaeger).
- **swift-log-otlp scope** if it ships as Tranche 3C — minimum viable is OTLP log envelope encoding; beyond that lies log severity mapping, log attribute conventions, etc.

## Migration impact

No migrations required. Phase 3 adds new packages; no existing v0.1+ package changes its API.

If we extract OTLP common types to a shared target (one of the unresolved questions), swift-otlp-exporter would need a 0.2 minor bump (pre-1.0; SemVer-permitted) to depend on the new shared target. This is the same pre-1.0 minor-bump pattern swift-prometheus 0.2 used; the precedent is set.

## Resolution

_(Filled by the project lead at merge time.)_

- **Accepted** / **Rejected** / **Withdrawn**
- Date:
- Notes:

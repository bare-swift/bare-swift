# RFC-0005 — Phase 2 anchor: observability wave

| Field | Value |
| --- | --- |
| Status | Draft |
| Author | bare-swift project lead |
| Created | 2026-05-07 |
| Resolution | _(filled at merge)_ |

## Summary

Anchor Phase 2 on the **observability wave**: swift-hdrhistogram, swift-statsd, swift-otlp-exporter, plus swift-prometheus 0.2 (Summary type + native histograms + OpenMetrics format) and a sibling swift-prometheus-metrics adapter for swift-metrics. Phase 2 also produces a Bytes RFC and a swift-bytes package as a prerequisite of the OTLP exporter.

This anchor builds directly on Tranche 1C (swift-ddsketch + swift-prometheus). Phase 1 audiences carry over; the technical surface is well-understood.

## Problem

The roadmap §7 Gate 1 → Phase 2 criteria require an "anchor decision recorded as an RFC" before Phase 2 plans begin. The roadmap §7 lists three candidate waves and says nothing about which to pick — that's this RFC's job.

Phase 1 shipped 10 small packages with relative-error guarantees on the heavy ones. The question now is: where does Phase 2's effort produce the most leverage?

## Proposal

### Anchor: observability wave

Phase 2 primary work targets observability — the same domain as Tranche 1C. Specifically:

- **swift-bytes** (after RFC-0006): a span-like byte buffer / view type. Foundation-free, allocation-aware. Required by swift-otlp-exporter for protobuf serialization without round-tripping through `[UInt8]`. Likely also informs Phase 3 config/serialization wave.
- **swift-hdrhistogram**: a Swift port of HdrHistogram (Gil Tene's high-dynamic-range histogram). Complements swift-ddsketch for users who want fixed-precision-per-decade rather than fixed-relative-error.
- **swift-statsd**: a StatsD-protocol client (Etsy / Datadog dialects). Wire-format only, no UDP transport — caller wires a `Datagram` sender. Pairs with swift-prometheus for environments that haven't migrated.
- **swift-otlp-exporter**: an OTLP (OpenTelemetry Protocol) exporter — produces protobuf-serialized spans / metrics. Caller wires the gRPC or HTTP transport. The most ambitious Phase 2 package and the one that motivates swift-bytes.
- **swift-prometheus 0.2**: amends swift-prometheus 0.1 to add Summary (CKMS or t-digest backed), native histograms (Prometheus 2.40+ exponential buckets), and OpenMetrics text-format extras (`# UNIT`, `_created` series).
- **swift-prometheus-metrics**: a separate sibling package that adapts swift-metrics's `MetricsFactory` to swift-prometheus 0.2's registry. Pulls in `swift-metrics` as a dependency; swift-prometheus itself stays standalone.

### Tranches

| Tranche | Packages | Estimated duration |
|---|---|---|
| 2A | swift-bytes, swift-hdrhistogram | ~6 weeks |
| 2B | swift-statsd, swift-otlp-exporter (depends on 2A's bytes) | ~6 weeks |
| 2C | swift-prometheus 0.2, swift-prometheus-metrics | ~4 weeks |

Total: ~4 months calendar, mirroring Phase 1's pace. Tranche order is dependency-driven: Bytes must land before OTLP, and the prometheus-0.2 amendment + metrics adapter come last so they can leverage anything we learn from the new packages.

### Per-package scope sketch (formal specs follow during Phase 2)

**swift-bytes.** A `Bytes` value type (or family of types — `Bytes`, `BytesView`) holding a `[UInt8]`-equivalent payload with Foundation-free Span-like semantics. Source crate references: `bytes` (https://crates.io/crates/bytes), but more importantly the Swift stdlib's evolving `RawSpan` story (we should not block on RawSpan stabilization; ship our own seam now). Out of scope for v0.1: zero-copy reslicing across allocator boundaries, IO integration. Driven by RFC-0006.

**swift-hdrhistogram.** Single-process HdrHistogram with `recordValue`, `recordValueWithCount`, `valueAtPercentile`, mergeability, bounded memory cost. Source crate reference: `hdrhistogram` (https://crates.io/crates/hdrhistogram). Foundation-free. Tradeoff vs swift-ddsketch: HdrHistogram gives fixed precision per decade (configurable); DDSketch gives fixed *relative* error — different guarantees, different best uses.

**swift-statsd.** Wire-format encoder for StatsD's text protocol, in both Etsy and Datadog dialect (the latter adds tags). No UDP transport — output is `[UInt8]` (or `Bytes`). Caller wires `sendto`. Out of scope: Datadog tracing UDP packets (different protocol).

**swift-otlp-exporter.** Build OTLP-protobuf payloads (metrics + traces) from in-memory metric and span snapshots. Pure encoder, no transport. Composes with the eventual swift-grpc / swift-http packages — those aren't in scope for Phase 2. Out of scope for v0.1: histogram_data export's full feature set (only counter, gauge, histogram, summary in v0.1).

**swift-prometheus 0.2.** Adds Summary (CKMS or t-digest backed; pick one and document why), native histograms (exponential buckets, the Prometheus 2.40+ wire format), OpenMetrics extras (`# UNIT`, `_created` lines). Backwards-compatible with v0.1 API; Summary is additive. CHANGELOG calls out v0.1 → v0.2 migration paths (which is "use the new types if you want them").

**swift-prometheus-metrics.** A `PrometheusMetricsFactory` conforming to `swift-metrics`'s `MetricsFactory`. Bridges Counter / Recorder / Timer into Prometheus 0.2 metric types. Sibling repo so swift-prometheus stays free of swift-metrics.

### Phase 2 prerequisites (must land before 2A starts)

1. **RFC-0006 — Bytes / span-like buffer type.** Decides whether we adopt `swift-nio.ByteBuffer`, wait for stdlib `RawSpan`, or roll our own. Until this RFC is accepted, swift-bytes can't begin.
2. **RFC-0007 — `@unchecked Sendable` justification standard** (medium priority — surfaced in Gate 1 retrospective, becomes more pressing as Phase 2 packages have more reference types).
3. **swift-greet bump** to demonstrate the macOS-15 path if any Phase 2 package opts in (validates the on-ramp described in RFC-0003).

### Alternatives considered

**Anchor on the config & serialization wave (TOML, YAML, MessagePack, CBOR, JSONPath).** Strong candidate. Rejected because:

- TOML and YAML are each substantially more work than any Phase 1 package (full parser plus full emitter, both with edge cases).
- JSONPath (RFC 9535) is its own large spec; it would be a tranche on its own.
- The wave's success hinges on Bytes anyway (efficient binary formats are the half of the case).
- Adoption story is more diffuse — every Swift project has its own preferred config format already; switching costs are high.
- Better fit for Phase 3 once Bytes has shipped and is battle-tested.

**Anchor on the routing & middleware wave (router, tower-equivalent, mime, cookie).** Rejected because:

- Middleware composition without a Bytes story is putting the cart before the horse.
- The Swift server-framework landscape (Hummingbird, Vapor, swift-http-types) is more fragmented than the metrics ecosystem; "build the next thing" is less obvious.
- Most attractive *after* observability is solid, because middleware that can't observe itself is incomplete.

**Mixed slate (one of each wave, smaller scope per).** Rejected because: mixing waves means three small adoption signals instead of one strong one. Phase 1's success was concentration — Tranche 1C explicitly bundled the adoption-magnet packages.

**Skip Phase 2; declare "done" at Phase 1.** Considered briefly. Rejected because the 10 Phase 1 packages by themselves don't compose into anything bigger; they're foundations. Building observability on top is what proves the foundations.

### What this anchor explicitly does NOT do

- It does **not** commit to a swift-tracing or swift-distributed-tracing dependency. swift-otlp-exporter accepts already-shaped `Span` data; integration with swift-distributed-tracing is a sibling package, not part of this anchor.
- It does **not** include an HTTP/gRPC transport layer. The exporter produces bytes; the caller chooses the wire.
- It does **not** include a process collector (CPU, memory, GC) or runtime collector. Those are heavily platform-specific and would benefit from being a separate, opt-in package.
- It does **not** ship a swift-metrics-default factory selection mechanism. The user explicitly opts into PrometheusMetricsFactory.

## Drawbacks

- **Adopter overlap.** Phase 1's swift-prometheus and swift-ddsketch already serve "Swift services that report metrics". Phase 2 doubles down on the same audience. If that audience is small, we hit diminishing returns. Mitigation: swift-statsd and swift-otlp-exporter open new niches (StatsD-only shops; OTLP-collector shops) that swift-prometheus alone doesn't reach.

- **OTLP scope creep.** OTLP's metric-data spec is large; getting Histogram / ExponentialHistogram / Summary all serialized correctly is real work. We mitigate by scoping v0.1 to Counter + Gauge + Histogram in protobuf form, deferring ExponentialHistogram and Summary to v0.2 (consistent with how we sequenced swift-prometheus 0.1 → 0.2).

- **Bytes RFC blocks the start.** The first ~2–3 weeks of Phase 2 are effectively "write RFC-0006, accept it, then begin 2A." That's design work, not shipping work. Phase 1's pattern was speed; Phase 2 starts slower by design.

- **swift-prometheus 0.2 is technically a v0.x → v0.x bump on an existing package.** Pre-1.0 SemVer permits API breaks across minor versions, but we should not break existing v0.1 APIs gratuitously. The Summary, native histograms, and OpenMetrics work are all *additive*; this drawback exists only if we discover we need to break v0.1 to add them — at which point we have a choice to make and a CHANGELOG to update.

## Unresolved questions

- Should swift-otlp-exporter live in its own repo (current proposal) or as a sub-target of swift-prometheus? Pro-separate: keeps swift-prometheus standalone. Pro-sub-target: avoids dual maintenance. Probably separate; defer the final call to the swift-otlp-exporter RFC.
- Should swift-statsd's Datadog dialect support be a feature flag (`#if datadog`) or always-on? Probably always-on; the wire format diverges at the tag level only, easily encoded.
- swift-distributed-tracing integration — leave for Phase 3 or include a tiny adapter as a stretch goal? Default: leave for Phase 3. swift-otlp-exporter's surface should permit adapters without dictating one.

## Migration impact

- **Phase 1 packages do not change.** swift-prometheus 0.1.x and swift-ddsketch 0.1.x continue to be maintained on their `main` branches independently of Phase 2 work. Bug fixes ship as 0.1.x patch releases.
- **swift-prometheus 0.2.0** is a minor bump; v0.1 users continue to compile against the unchanged v0.1 surface. Migration is opt-in (use the new Summary type, native histograms, etc.). CHANGELOG documents the v0.1 → v0.2 path.
- **swift-bytes** is new; no migration. Future packages that want a span-like buffer adopt it from day one.
- **swift-prometheus-metrics** is new; users currently using their own swift-metrics ↔ Prometheus glue can migrate by replacing their factory with `PrometheusMetricsFactory`.

## Resolution

_(Filled by the project lead at merge time.)_

- **Accepted** / **Rejected** / **Withdrawn**
- Date:
- Notes:

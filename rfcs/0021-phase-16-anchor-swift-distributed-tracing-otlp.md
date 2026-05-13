# RFC-0021 — Phase 16 anchor: swift-distributed-tracing-otlp adapter

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-13 |
| Resolution | Accepted 2026-05-13 — Phase 16 plans may begin authoring. Single new package: swift-distributed-tracing-otlp. |

## Summary

Anchor Phase 16 on **swift-distributed-tracing-otlp** — a new adapter package that bridges Apple's [swift-distributed-tracing](https://github.com/apple/swift-distributed-tracing) into the bare-swift OTLP exporter family (swift-tracing-otlp + swift-log-otlp + swift-otlp-exporter). Closes Phase 14's documented "implicit context bridge" gap by wiring `ServiceContext.current` (swift-distributed-tracing's task-local) to `OTLP.TraceContext` (swift-tracing-otlp v0.3's W3C value type).

Single-anchor phase. Structurally similar to Phase 3 (swift-tracing-otlp v0.1) and Phase 10 (swift-brotli v0.1) — one focused new package built on top of stable existing primitives.

## Problem

[Gate 15 retrospective](../docs/gates/2026-05-13-gate-15-retrospective.md) item 5 requires "Phase 16 anchor decision recorded as an RFC" before Phase 16 plans begin. The retrospective surveyed seven candidate waves (swift-distributed-tracing-otlp adapter, swift-jwt-verify v0.2 signing, swift-idna v0.3 Bidi+ContextJ, swift-brotli v0.3 streaming, OAuth 2.0, crypto-adjacent, swift-publicsuffix v0.2); this RFC formalizes the choice.

**Concrete gap from Phase 14:** RFC-0019 originally sketched implicit-context APIs on swift-tracing-otlp v0.3 and swift-log-otlp v0.3 — `TraceContext.current` reading the active swift-distributed-tracing `ServiceContext`, and a swift-log LogHandler that auto-attaches trace correlation fields. Tranches 14B/14C reshaped to **explicit-input APIs** (the right shape for encoder packages, decoupled from any specific logging/tracing frontend) but punted the implicit-context wiring to a future adapter package.

Today, an adopter using:
- `swift-distributed-tracing` for span management (Apple)
- `swift-log` for structured logging (Apple)
- `swift-tracing-otlp` for OTLP/HTTP/protobuf trace export (bare-swift Phase 3 + 14)
- `swift-log-otlp` for OTLP/HTTP/protobuf log export (bare-swift Phase 4 + 14)

has to manually plumb `TraceContext` through their application code: capture the active `ServiceContext` at every log site, extract `traceID` / `spanID` / `flags`, build an `OTLP.TraceContext`, pass it to `OTLP.LogRecord(traceContext: ...)`. This is a five-line ceremony at every log site.

The package that *should* exist:

```swift
import DistributedTracingOTLP    // new in Phase 16

// One-time wiring at app startup:
LoggingSystem.bootstrap { label in
    OTLPLogHandler(label: label, exporter: myOTLPLogExporter)
}
InstrumentationSystem.bootstrap(OTLPTracer(exporter: myOTLPTraceExporter))

// At every log site — no ceremony:
logger.info("user logged in", metadata: ["user.id": "\(userID)"])
// The handler captures ServiceContext.current, extracts trace IDs,
// auto-attaches them to the emitted OTLP.LogRecord.
```

After Phase 14, all the primitives exist. Phase 16 wires them together.

**Why this is the right scope as an adapter package, not extending swift-tracing-otlp or swift-log-otlp:**

1. Gate 14 retro identified this explicitly: encoder packages get explicit-input APIs; implicit-context wiring belongs in adapter packages. Mixing the two violates the separation.
2. Apple's swift-distributed-tracing introduces a hard dependency that not every swift-tracing-otlp / swift-log-otlp adopter wants. Keeping the dependency in an adapter package lets the encoder packages stay frontend-agnostic.
3. The adapter is the natural composition point: it pulls in swift-distributed-tracing + swift-distributed-tracing-bridge + swift-tracing-otlp + swift-log-otlp, and exports a single ergonomic surface to apps.

After Phases 3 / 4 / 7 / 8 / 9 / 10 / 11 / 12 / 13 / 14 / 15, the bare-swift observability tier has all the primitives but no opinionated adapter. Phase 16 closes that gap.

## Proposal

### Anchor: swift-distributed-tracing-otlp (single new package)

**swift-distributed-tracing-otlp v0.1** — adapter wiring Apple's swift-distributed-tracing into bare-swift's OTLP exporter family. Module name: `DistributedTracingOTLP`. Repo: `bare-swift/swift-distributed-tracing-otlp`.

| API | Purpose |
|---|---|
| `OTLPLogHandler` | swift-log `LogHandler` that captures `ServiceContext.current` on each log call, extracts trace correlation fields via `OTLP.TraceContext`, and emits `OTLP.LogRecord` via a caller-provided exporter closure. |
| `OTLPTracer` | swift-distributed-tracing `Tracer` (and `LegacyTracer`) that builds `OTLP.Span` values on `Span.end()` and emits them via a caller-provided exporter closure. |
| `OTLP.TraceContext.current` | TaskLocal accessor reading the active swift-distributed-tracing `ServiceContext` and returning the corresponding `OTLP.TraceContext` (or `nil` if no active context). |
| `DistributedTracingOTLPError` | Typed-throws enum for failure modes during context extraction / exporter wiring. |

### Tranches

Single-anchor phase shape, two possible decompositions:

**Shape A (recommended): single tranche, full v0.1**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 16A | swift-distributed-tracing-otlp v0.1 | `OTLPLogHandler` + `OTLPTracer` + `OTLP.TraceContext.current` accessor + DistributedTracingOTLPError | ~1.5 days |

**Shape B (alternate): two tranches if Instrument scope expands**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 16A | swift-distributed-tracing-otlp v0.1 | `OTLPLogHandler` + `OTLP.TraceContext.current` (LogHandler-only) | ~1 day |
| 16B | swift-distributed-tracing-otlp v0.2 | `OTLPTracer` (`Tracer` + `LegacyTracer` protocol implementation; OTLP span emission on `Span.end()`) | ~0.5 day |

Decision between A and B is deferred to the brainstorm phase, which must verify the Apple swift-distributed-tracing `Tracer` protocol shape against the reference implementation (similar pattern to Phase 12's swift-brotli encoder LOC reality-check). The likely outcome based on prior phase patterns: Shape B emerges if the `Tracer` protocol surface is wider than expected, allowing 16A to ship as the smaller LogHandler-only tranche.

### Tranche 16A scope sketch

`swift-distributed-tracing-otlp` v0.1 covers:

- **`OTLP.TraceContext.current` TaskLocal accessor.** Reads `ServiceContext.current` via swift-distributed-tracing, extracts the `SpanContext` (trace ID + span ID + trace flags), constructs an `OTLP.TraceContext`. Returns `nil` if no active context.
- **`OTLPLogHandler`** conforming to `LogHandler` from swift-log. On each log call: capture `OTLP.TraceContext.current`, build an `OTLP.LogRecord` using the v0.3 convenience init `OTLP.LogRecord(timeUnixNano:, severityNumber:, body:, traceContext: ctx)`, hand the record to a caller-provided exporter closure (typically a wrapper around `OTLP.encodeLogs(_:)` from swift-log-otlp).
- **`OTLPTracer`** conforming to `Tracer` from swift-distributed-tracing. `startSpan(_:context:ofKind:at:function:file:line:)` creates an `OTLPSpan` (internal type) that tracks attributes/events/links. `Span.end()` builds an `OTLP.Span` and hands it to a caller-provided exporter closure (typically a wrapper around `OTLP.encodeTraces(_:)` from swift-tracing-otlp).
- **`DistributedTracingOTLPError`** typed-throws enum with cases for the documented failure modes.

**Reuses existing types:**
- `OTLP.TraceContext` (swift-tracing-otlp v0.3)
- `OTLP.Span` / `OTLP.Span.Event` / `OTLP.Span.Link` / `OTLP.Status` (swift-tracing-otlp v0.1+)
- `OTLP.LogRecord` (swift-log-otlp v0.1+) with `traceContext:` convenience init (v0.3)
- `OTLP.SpanContext` / `OTLP.SpanID` / `OTLP.TraceID` (swift-distributed-tracing-bridge v0.1)

**Public API uses non-throwing functions where possible.** Logging and span emission are total operations on value types; the only throwing entry points are the optional exporter closures (which throw whatever the adopter's HTTP layer throws).

**Out of scope for v0.1:**
- **gRPC OTLP transport.** Stay HTTP+protobuf (matches all four existing OTLP packages).
- **HTTP client wiring.** The exporter closures take `Bytes` and the caller wires to URLSession / async-http-client / NIO. Pure adapter; no transport.
- **Sampling.** Implicit "sample everything" until a sampler is plumbed; sampler integration deferred to v0.2 if adopter demand surfaces.
- **Batch / buffer / retry.** The exporter closure is called on every span/log; adopters can wrap with their own batching.
- **Cross-process context propagation.** The W3C `traceparent` header (swift-tracing-otlp v0.3) is already first-class; this package wires task-local context, not HTTP-level propagation.

### Tranche 16B scope sketch (alternate / stretch)

If 16A scope-cuts to LogHandler-only based on brainstorm findings:

`swift-distributed-tracing-otlp` v0.2 adds:
- `OTLPTracer` conforming to swift-distributed-tracing's `Tracer` protocol.
- Internal `OTLPSpan` type tracking attributes / events / links / status during a span's lifetime.
- On `Span.end()`, build an `OTLP.Span` and emit via the exporter closure.

~150-250 LOC additive. All v0.1 tests still pass.

### Why this anchor

1. **Closes the longest-standing Phase-14 design gap.** The "implicit context bridge" punted from RFC-0019 needs to land in a package. swift-distributed-tracing-otlp is the natural home.
2. **Net-new package balances the v0.x churn.** Phase 15 did two BREAKING bumps on existing packages (swift-idna v0.1 → v0.2, swift-uri v0.2 → v0.3). Phase 16 should give adopters a stable phase on the existing packages while introducing a new opt-in surface.
3. **Audience continuity with two communities.** swift-distributed-tracing adopters (server-side Swift) plus OTLP exporter adopters. Both communities have grown out of bare-swift's observability tier.
4. **Risk profile is well-bounded.**
   - swift-distributed-tracing's `Tracer` / `LegacyTracer` protocols are stable (Apple's package, well-versioned).
   - swift-log's `LogHandler` protocol is stable.
   - All OTLP message types exist; the adapter just wires the protocols.
   - The trickiest piece is the `Span.end()` → `OTLP.Span` conversion, which requires tracking attributes/events/links during the span's lifetime. Reference implementations exist in opentelemetry-swift.
5. **Sets up Phase 17+ options.**
   - **swift-jwt-verify v0.2 signing** has had a week to settle.
   - **swift-idna v0.3 (Bidi + ContextJ)** still available.
   - **swift-brotli v0.3 streaming** still available.
   - **OAuth 2.0** still available.

### Out of scope for Phase 16

- **swift-jwt-verify v0.2 signing.** Phase 17+ candidate.
- **swift-idna v0.3 (Bidi + ContextJ + ContextO).** Phase 17+ candidate.
- **swift-brotli v0.3 streaming encoder.** Phase 17+ candidate.
- **OAuth 2.0 / crypto-adjacent.** Same standing rejections.
- **swift-publicsuffix v0.2 (ICANN/PRIVATE split).** Quiet patch when an adopter asks.
- **gRPC OTLP transport** in any package. Separate spec; not in v0.1 scope.

### Tooling expectations

Phase 16 reuses all Phase 7–15 tooling without extension:

- `bare-swift new` for swift-distributed-tracing-otlp scaffolding.
- Pre-emptive Pages tag-policy on the new repo (per [[feedback-github-pages-new-repo]]).
- `bare-swift gen-site --umbrella .` after the v0.1 release.
- Strict-concurrency in CI stays mandatory.
- `docc-target` opt-in from scaffold time (Gate 11 lesson).
- Sanitizers ON (no large static data expected in v0.1).
- Cross-library validation at ship time, NOT in test target. For this package: round-trip `ServiceContext` → `OTLP.TraceContext` → bytes via swift-tracing-otlp / swift-log-otlp, decode with the reference opentelemetry-collector locally.

### Versioning

| Package | New | Future |
|---|---|---|
| swift-distributed-tracing-otlp | v0.1 (new — LogHandler + Tracer) | v0.2 = sampler integration if adopter demand; v0.3 = batch/buffer/retry; v0.4 = gRPC if Phase 18+ adds it |

Phase 16 introduces **1 new package**, taking the ecosystem from **46 to 47 packages**.

### Anti-goals

- **No gRPC.** Out of scope; separate OTLP spec.
- **No HTTP client.** Pure adapter; callers wire to their HTTP client.
- **No retroactive changes** to swift-tracing-otlp / swift-log-otlp / swift-otlp-exporter / swift-distributed-tracing-bridge. Phase 16 is purely additive on top.
- **No sampler.** v0.1 emits everything; sampling integration is v0.2+ if demand surfaces.
- **No new umbrella tier.** The package fits the existing `observability` tier.

## Alternatives considered

### Anchor Phase 16 on swift-jwt-verify v0.2 (signing + RS256/ES384/EdDSA)

**Pros:** closes the verify-only design boundary; rounds out the JWT story for issuers.

**Cons:** v0.1 (Phase 11) shipped only ~1 day before Phase 15 started. Real adopter signal from production verifier usage hasn't accumulated. Multi-algorithm scope is wide (RS256 vs ES384 vs EdDSA via swift-crypto have different surface areas).

**Verdict:** rejected as Phase 16. Strong Phase 17+ candidate when v0.1 has had a week of adopter use.

### Anchor Phase 16 on swift-idna v0.3 (Bidi + ContextJ + ContextO)

**Pros:** closes Phase 15's documented deferral cleanly; small focused scope (~500 LOC).

**Cons:** three consecutive v0.x bumps on the same package in three phases would be aggressive churn for adopters. The Bidi rule and ContextJ checks are rarely-hit in real-world hostname data; no adopter demand surfaced yet. Phase 17+ when demand surfaces.

**Verdict:** rejected as Phase 16. Phase 17+ candidate.

### Anchor Phase 16 on swift-brotli v0.3 (streaming encoder)

**Pros:** closes the v0.2 one-shot-only deferral; streaming is useful for large HTTP responses.

**Cons:** narrower audience than OTLP cross-signal. swift-brotli v0.2 shipped 2 days ago — adopter signal hasn't accumulated.

**Verdict:** rejected as Phase 16. Phase 17+ candidate.

### Anchor Phase 16 on OAuth 2.0 client primitives

**Pros:** composes naturally on top of swift-bearer + swift-jwt-verify.

**Cons:** still no concrete adopter demand. OAuth 2.0 has many flows; picking the right v0.1 subset needs adopter input.

**Verdict:** rejected as Phase 16. Defer until concrete demand.

### Skip ahead, do crypto-adjacent (swift-blake3 / swift-siphash / swift-hkdf)

**Pros:** would complement swift-jwt-verify if signing ever lands.

**Cons:** swift-crypto remains entrenched. **12th consecutive rejection** across gates.

**Verdict:** rejected.

### Anchor Phase 16 on swift-publicsuffix v0.2 (ICANN/PRIVATE split)

**Pros:** closes a quiet Phase 13 deferral.

**Cons:** narrow scope. Better as a v0.1.1 patch when an adopter needs it. A full Phase 16 anchor would be over-rotating on a small feature.

**Verdict:** rejected as Phase 16 anchor. Will land as a quiet patch when demand surfaces.

## Resolution

**Accepted 2026-05-13.** Phase 16 plans may begin authoring. Single new package: swift-distributed-tracing-otlp.

Phase 16 closes the Phase 14 design gap (implicit-context bridge between swift-distributed-tracing and the OTLP exporter family). After Phase 16, an adopter can run swift-distributed-tracing + swift-log + bare-swift OTLP packages with **zero per-call-site plumbing** — the adapter captures the active context and threads it through automatically.

swift-jwt-verify v0.2 signing, swift-idna v0.3 (Bidi + ContextJ), swift-brotli v0.3 streaming, and OAuth 2.0 remain on the queue for Phase 17+.

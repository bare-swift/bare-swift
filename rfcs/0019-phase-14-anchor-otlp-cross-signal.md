# RFC-0019 — Phase 14 anchor: OTLP cross-signal v0.2

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-13 |
| Resolution | Accepted 2026-05-13 — Phase 14 plans may begin authoring. First package: swift-otlp-json (new). |

## Summary

Anchor Phase 14 on the **OTLP cross-signal wave**: ship swift-otlp-json (new package — OTLP/HTTP/JSON export for traces + logs + metrics) and bump swift-otlp-traces + swift-otlp-logs to v0.2 with cross-signal correlation (TraceContext IDs in log records, exemplars in metric points). Three tranches, structurally similar to Phase 11 (auth primitives). Closes Phase 3+4's single-signal deferral on the observability tier.

The wave decomposes by independence: swift-otlp-json is a self-contained new package; the v0.2 bumps are additive on existing packages.

## Problem

[Gate 13 retrospective](../docs/gates/2026-05-13-gate-13-retrospective.md) item 5 requires "Phase 14 anchor decision recorded as an RFC" before Phase 14 plans begin. The retrospective surveyed six candidate waves (OTLP cross-signal, swift-idna v0.2, swift-jwt-verify v0.2 signing, swift-brotli v0.3 streaming, OAuth 2.0, crypto-adjacent); this RFC formalizes the choice.

OTLP cross-signal has been the standing #2 candidate since Gate 11 (after i18n). With Phase 13 closed, OTLP is now the leading candidate. Phase 3 (swift-otlp-traces v0.1) and Phase 4 (swift-otlp-logs v0.1) both shipped as single-signal exporters — they emit valid OTLP messages but don't correlate across signals. Production observability requires cross-signal correlation: a span gets a trace ID, then logs emitted within that span carry the trace+span IDs, and metric exemplars link to the trace span that produced them.

The concrete gap: a swift-distributed-tracing adopter exporting via swift-otlp-traces can't currently emit logs (via swift-log + swift-otlp-logs) that link back to the active span. Correlation requires application-level wiring that should be automatic per the W3C TraceContext spec.

**Additional gap:** swift-otlp-traces and swift-otlp-logs only export protobuf. OTLP/HTTP/protobuf is the most common transport, but OTLP/HTTP/JSON exists for cases where protobuf is impractical (e.g., debug serialization, JSON-only telemetry backends). swift-otlp-json adds this transport.

After Phases 4 / 7 / 8 / 9 / 10 / 11 / 12 / 13, the bare-swift observability tier provides per-signal export (traces, logs, metrics) but lacks cross-signal correlation. Phase 14 closes it.

## Proposal

### Anchor: OTLP cross-signal wave (1 new package + 2 v0.2 bumps)

- **swift-otlp-json (new v0.1)** — OTLP/HTTP/JSON export adapter. Mirrors swift-otlp-traces / -logs / -metrics protobuf exporters but emits JSON-encoded OTLP messages per OTLP spec § 6 (HTTP/JSON encoding). ~600 LOC. swift-json is in-house; output is canonical JSON matching the protobuf wire form schema.
- **swift-otlp-traces v0.2** — add TraceContext propagation hooks so adopters can extract `traceId` + `spanId` for use in log records / metric exemplars. ~200 LOC. swift-distributed-tracing-bridge wires these into the active task context.
- **swift-otlp-logs v0.2** — extract active-trace-context (traceId + spanId) and attach to emitted log records per OTLP spec § 7.4 (`trace_id` + `span_id` fields on `LogRecord`). ~200 LOC.

Total: **~1000 LOC source** across three packages. Comparable in shape to Phase 11 (auth primitives, ~700 LOC) and Phase 12 (brotli encoder, ~1700 LOC).

### Tranches

| Tranche | Packages | Estimated calendar |
|---|---|---|
| 14A | swift-otlp-json v0.1 (new package — OTLP/HTTP/JSON encoder) | ~1.5 days |
| 14B | swift-otlp-traces v0.2 (TraceContext propagation hooks) | ~0.5 day |
| 14C | swift-otlp-logs v0.2 (trace_id + span_id on LogRecord) | ~0.5 day |

Total Phase 14 budget: ~2.5 days wall-clock at observed pace. Resembles Phase 11 (auth primitives) in shape — multiple small-to-medium tranches sharing an audience.

**Optional reshape:** 14A could be the single anchor with 14B/14C as optional follow-ons (Phase 10 / Phase 12 shape). Decision deferred to the implementation plan.

### Why this anchor

1. **Two-gate-deep consensus.** Gate 11 + Gate 12 both named OTLP cross-signal as the next-strongest signal after i18n. With Phase 13 closed, OTLP is the leading candidate.
2. **Closes the longest-standing v0.x deferral in the observability tier.** swift-otlp-traces shipped v0.1 in Phase 3 (2026-05-09); swift-otlp-logs shipped v0.1 in Phase 4 (2026-05-10). Both have been single-signal for ~10 phases.
3. **Audience continuity.** Same observability folks who use swift-prometheus / swift-prometheus-metrics / swift-distributed-tracing-bridge. The HTTP-server stack closed Phase 11; observability is the other major adopter community.
4. **Decomposes naturally into 3 tranches.** swift-otlp-json is a self-contained new package; the v0.2 bumps are additive on existing packages. Independent shipping order: 14A → 14B → 14C unblocks adopters incrementally.
5. **Risk profile is well-bounded.**
   - swift-otlp-json (v0.1, new): swift-json is in-house; OTLP JSON schema is specified in the OTLP repo with test vectors. ~600 LOC + reuse of existing message types from swift-otlp-traces / -logs.
   - swift-otlp-traces v0.2: additive — adds `TraceContext.current` accessor + W3C trace-context header helpers. ~200 LOC. All v0.1 tests still pass.
   - swift-otlp-logs v0.2: additive — attaches trace+span IDs to log records when a swift-distributed-tracing context is active. ~200 LOC. All v0.1 tests still pass.
6. **Sets up Phase 15+ options.** Once OTLP cross-signal ships:
   - **swift-idna v0.2 (UTS #46 + NFC)** becomes the natural Phase 15 anchor — closes Phase 13's deferral.
   - **swift-jwt-verify v0.2 signing** stays available.
   - **swift-brotli v0.3 streaming** stays available.
   - **OAuth 2.0** stays available.

### Tranche 14A scope sketch

`swift-otlp-json` v0.1 covers OTLP/HTTP/JSON export per the OTLP specification's § 6 (HTTP/JSON encoding):

- `OTLPJSON.encode(_ request: ExportTraceServiceRequest) -> String` — serialize a traces export request to canonical OTLP/HTTP/JSON.
- `OTLPJSON.encode(_ request: ExportLogsServiceRequest) -> String` — logs.
- `OTLPJSON.encode(_ request: ExportMetricsServiceRequest) -> String` — metrics.
- `OTLPJSONError` typed-throws enum (validation errors during encoding).
- Reuses the OTLP message types declared in swift-otlp-traces / -logs / -metrics — depends on all three plus swift-json.

**Out of scope for v0.1:**
- HTTP client wiring (callers send the JSON via swift-nio or URLSession). swift-otlp-json is encoder-only.
- gRPC encoding (out of scope; OTLP-over-gRPC is a separate spec).
- Compression (callers handle Content-Encoding via swift-content-encoding).

### Tranche 14B scope sketch

`swift-otlp-traces` v0.2 adds TraceContext propagation hooks:

- `TraceContext.current` — read the currently-active swift-distributed-tracing `ServiceContext`'s trace+span IDs.
- W3C trace-context header builders: `TraceContext.traceparent(_:)` / `TraceContext.tracestate(_:)`.
- All v0.1 export APIs unchanged. Additive only.

### Tranche 14C scope sketch

`swift-otlp-logs` v0.2 attaches trace+span IDs to log records:

- Public `LogRecord.traceID` / `.spanID` fields (optional `[UInt8]` / `[UInt8]`).
- The swift-log → OTLP exporter automatically reads `TraceContext.current` when emitting log records, attaching IDs when present.
- All v0.1 export APIs unchanged. Additive only.

### Out of scope for Phase 14

- **OTLP/HTTP/protobuf transport.** Already exists in swift-otlp-traces / -logs. Phase 14 adds JSON alongside, not replacing.
- **OTLP/gRPC transport.** Separate spec; not in v0.1 scope.
- **HTTP client wiring.** swift-otlp-json is encoder-only; callers wire to their HTTP client.
- **swift-idna v0.2 UTS #46.** Phase 15 anchor candidate.
- **swift-jwt-verify v0.2 signing.** Phase 15+.
- **OAuth 2.0 / swift-brotli v0.3 streaming.** Phase 15+.

### Tooling expectations

Phase 14 reuses all Phase 7–13 tooling without extension:

- `bare-swift new` for swift-otlp-json scaffolding.
- `bare-swift gen-site --umbrella .` after each v0.x release.
- Pre-emptive Pages tag-policy on the new repo.
- Strict-concurrency in CI stays mandatory.
- `docc-target` opt-in from scaffold time (mandatory from Gate 11).
- Sanitizers ON (no large static data expected in v0.1).
- Cross-library validation at ship time, NOT in test target. For swift-otlp-json: round-trip JSON output through the OTLP spec's reference test vectors.

### Versioning

| Package | New | Future |
|---|---|---|
| swift-otlp-json | v0.1 (encoder-only; traces + logs + metrics) | v0.2 = JSON-Lines streaming if adopter demand surfaces |
| swift-otlp-traces | v0.2 (TraceContext hooks) | v0.3 if cross-signal patterns need more API surface |
| swift-otlp-logs | v0.2 (trace_id + span_id on LogRecord) | v0.3 if structured-log-record patterns need more |

Phase 14 introduces **1 new package**, taking the ecosystem from **45 to 46 packages**. Plus 2 version bumps on existing packages.

### Anti-goals

- **No OTLP/gRPC.** Out of scope; separate spec.
- **No HTTP client.** Encoder-only; callers wire.
- **No changes to existing v0.1 export APIs.** All v0.1 consumers continue to work.
- **No retroactive bug-fix sweep across prior phases.**

## Alternatives considered

### Anchor Phase 14 on swift-idna v0.2 (UTS #46 + NFC)

**Pros:** closes Phase 13's documented deferral; needed before swift-uri v0.3 auto-encode.

**Cons:** large Unicode tables (~200 KiB packed) + spec complexity (UTS #46 mapping + NFC + Bidi rule + ContextJ/O). ~3-5k LOC. Deserves its own phase as a single-anchor, similar to Phase 10 (brotli decoder) and Phase 12 (brotli encoder). Phase 13 deferral isn't aging fast yet (no adopter complaints).

**Verdict:** rejected as Phase 14. Strong Phase 15 single-anchor candidate.

### Anchor Phase 14 on swift-jwt-verify v0.2 (signing + RS256)

**Pros:** closes the verify-only design boundary; completes the JWT story.

**Cons:** swift-jwt-verify v0.1 shipped in Phase 11 (2026-05-12 — three days ago). Let v0.1 settle and gather adopter signal before expanding scope.

**Verdict:** rejected as Phase 14. v0.2 minor release if adopter demand surfaces.

### Anchor Phase 14 on swift-brotli v0.3 (streaming encoder)

**Pros:** closes a v0.2 deferral (one-shot only); streaming is useful for large HTTP responses.

**Cons:** narrower audience than OTLP. The v0.2 one-shot encoder shipped 2 days ago and is unlikely to be hitting size limits in production yet.

**Verdict:** rejected as Phase 14. Phase 15+ candidate.

### Anchor Phase 14 on OAuth 2.0 client primitives

**Pros:** composes naturally on top of swift-bearer + swift-jwt-verify.

**Cons:** no concrete adopter demand. OAuth 2.0 has many flows; picking the right v0.1 subset needs adopter input.

**Verdict:** rejected as Phase 14. Defer until concrete demand.

### Skip ahead, do crypto-adjacent (swift-blake3 / swift-siphash / swift-hkdf)

**Pros:** would complement swift-jwt-verify if signing ever lands.

**Cons:** swift-crypto remains entrenched. **10th consecutive rejection** across gates. Differentiation hasn't strengthened.

**Verdict:** rejected.

## Resolution

**Accepted 2026-05-13.** Phase 14 plans may begin authoring. First package: swift-otlp-json (new).

Phase 14 closes the longest-standing v0.x deferral in the observability tier (single-signal exporters since Phases 3+4) and adds the OTLP/HTTP/JSON transport. After Phase 14, the bare-swift observability story supports: per-signal export (Phases 2-4) → cross-signal correlation (Phase 14) → JSON transport (Phase 14) — the full triplet that observability backends expect.

swift-idna v0.2 UTS #46, swift-jwt-verify v0.2 signing, swift-brotli v0.3 streaming, and OAuth 2.0 remain on the queue for Phase 15+.

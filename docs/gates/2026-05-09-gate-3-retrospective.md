# Gate 3 Retrospective: Phase 3 → Phase 4

**Date:** 2026-05-09
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 3 (the distributed tracing wave anchored by [RFC-0007](../../rfcs/0007-phase-3-anchor-distributed-tracing.md)) against the Gate 1/Gate 2 criteria template, and recommends whether Phase 4 should start. Tranche 3C (swift-log-otlp) was deliberately skipped — see Item 2.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | Anchored Tranche 3A/3B packages shipped at v0.1+, CI green, DocC live, listed | ✓ PASS |
| 2 | RFC-0007 commitments fulfilled (OTel metrics+traces complete; 3C logs deferred by design) | ✓ PASS |
| 3 | ≥1 new RFC accepted during Phase 3 (the closing gate's anchor counts; RFC-0007 itself accepted at start) | partial — see below |
| 4 | RFC-0001 conventions stress-test against Phase 3 packages | ✓ DONE BELOW |
| 5 | Phase 4 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0008 below) |

**Overall:** Items 1, 2, 4 pass cleanly. Item 3 partially passes (RFC-0007 was accepted at the *start* of Phase 3 to satisfy Gate 2; no further RFCs landed during execution — same shape as Phase 2). Item 5 is open. Phase 4 plans should be drafted but not executed until item 5 closes via RFC-0008.

The roadmap's stop conditions did not trigger: Phase 3 calendar time was ~2 days (vs the 3–6 month original budget — same pattern as Phase 2; the AI-assisted execution model is sustainably faster, not corner-cutting). No security incidents. No Apple announcements that subsumed planned packages. RFC-0001 held under the second non-bare-swift dep stress test.

---

## Item 1 — Phase 3 packages shipped ✓ PASS

| Tranche | Package | v0.1+ tag | DocC | Index | CI |
|---|---|---|---|---|---|
| 3A | swift-tracing-otlp | v0.1.0 | ✓ | ✓ | green |
| 3B | swift-distributed-tracing-bridge | v0.1.0 | ✓ | ✓ | green |
| 3C *(stretch)* | swift-log-otlp | — | — | — | deliberately skipped |

Two of three Tranches shipped; Tranche 3C was explicitly tagged as a stretch deliverable in RFC-0007 and is being skipped in favor of advancing to Phase 4. The OTel logs signal can be picked up at any time as a small Phase 4-or-later side deliverable; deferring it now does not block any current adopter.

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists 18 packages (10 from Phase 1, 5 from Phase 2, 2 from Phase 3, plus swift-greet from Phase 0).

**Calendar time:** 2026-05-08 (swift-tracing-otlp) through 2026-05-09 (swift-distributed-tracing-bridge) — 1–2 days. Inside the 3–6 month original budget by orders of magnitude. Test coverage is full (95 tests across two packages), CI green, DocC live, and end-to-end consumer integration verified for both.

---

## Item 2 — RFC-0007 commitments ✓ PASS

[RFC-0007](../../rfcs/0007-phase-3-anchor-distributed-tracing.md) committed to the distributed tracing wave with logs as a stretch goal:

| RFC-0007 commitment | Outcome |
|---|---|
| Tranche 3A: swift-tracing-otlp (OTLP traces encoder) | ✓ shipped |
| Tranche 3B: swift-distributed-tracing-bridge (Apple-frontend adapter) | ✓ shipped |
| Tranche 3C *(stretch)*: swift-log-otlp | deferred per design — RFC-0007 explicitly says "If not, defer to Phase 4" |
| Reuses ~80% of swift-otlp-exporter machinery | ✓ verified — ProtoWriter and Common encoders ported with byte-identical wire output; ~250 LOC duplication accepted |
| Adapter pattern carries from swift-prometheus-metrics | ✓ verified — same factory + buffer + caller-driven flush shape |
| Same Foundation-free guarantee (RFC-0001) | ✓ preserved across both packages |

The RFC's "unresolved questions" resolved as follows:

| RFC-0007 unresolved question | Resolution |
|---|---|
| Shared internal target vs. cross-package re-use for OTLP common types | Resolved: depend on swift-otlp-exporter, reuse public common types, duplicate internal ProtoWriter + Common encoders. swift-otlp-common extraction rejected as premature abstraction. Cleanest path. |
| Span buffering strategy | Resolved: caller-driven `flushExport()`. No background flusher in v0.1. |
| Sampling policy | Deferred to v0.2 (out of scope confirmed). |
| Baggage propagation formats | v0.1 ships custom `OTLPTraceIDsKey` only; W3C TraceContext / B3 / Jaeger deferred. |
| swift-log-otlp scope if it ships | N/A — Tranche 3C skipped. |

Notable findings beyond the RFC's stated commitments:

- **First package depending on four upstream packages** (3 bare-swift + 1 Apple). The dependency-pinning convention is implicit but works: pin all four in Package.swift, transitive resolution is clean.
- **Second non-bare-swift dep** added (Apple's swift-distributed-tracing) without violating RFC-0001. swift-service-context (its only transitive dep) is also Foundation-free.
- **Three new Apple-adapter gotchas** discovered (Mutex non-copyability with `let buffer = self.buffer` capture; `nonmutating set` illegal on class properties; `SpanEvent` init signature is `at instant:` not field-named; TracerInstant requires Comparable+Hashable+Sendable). Recorded as `feedback_swift_distributed_tracing` memory for future Apple-adapter packages.

---

## Item 3 — ≥1 new RFC accepted during Phase 3 — partial

**Pre-Phase 3 (Gate 2 closeout):** RFC-0007 (Phase 3 anchor) accepted 2026-05-08 to clear Gate 2 item 5.

**During Phase 3 execution:** zero new RFCs accepted.

Same partial-pass pattern as Phase 2. The RFC process produces decisions at gate-closeout time (anchoring the *next* phase) rather than mid-execution. This is operationally the right cadence — execution is mechanical once an anchor RFC exists; mid-execution RFCs would either be premature (no rough edges yet) or scope-creep.

**Candidates surfaced during Phase 3 execution that warrant RFCs (next gate's work):**

### RFC-0008 — Phase 4 anchor [the gating RFC for closing Gate 3]

Same shape as RFC-0005 / RFC-0007. Decision space mapped in item 5 below.

### RFC-0009 — Apple-adapter package conventions [low-medium priority]

**Trigger:** Two Apple-frontend adapters (swift-prometheus-metrics, swift-distributed-tracing-bridge) shipped with the same shape — final class, factory + buffer + caller-driven flush, ServiceContextKey for state, `[self]` capture in closures, no `nonmutating set`. The pattern is implicit but not documented.

**Proposal:** Codify the adapter convention as an RFC. Future adapters (logs adapter, swift-server-context adapter, etc.) follow the same shape.

### RFC-0010 — Inter-package code-sharing policy [low priority]

**Trigger:** swift-tracing-otlp duplicated swift-otlp-exporter's internal `ProtoWriter` + `EncodeCommon` (~250 LOC) instead of extracting a common package. The decision was correct (avoided premature abstraction) but is an informal policy. RFC would document: when to duplicate, when to extract, and what counts as "third user" triggering extraction.

### Recommendation

Author and accept **RFC-0008** (Phase 4 anchor) as the next governance work. That alone clears the spirit of item 3.

RFC-0009 / RFC-0010 are low-stakes; either bundle into a single "Phase 3 lessons" RFC, defer to Phase 4 mid-flight if relevant, or skip — the patterns are already in memory and being applied.

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

Per-package review against RFC-0001 for the 2 Phase 3 packages:

| Convention | Both | Notes |
|---|---|---|
| Module name PascalCase, no `Swift` prefix | ✓ | `TracingOTLP`, `DistributedTracingBridge` |
| Apache-2.0 + LLVM exception SPDX | ✓ | Every source file. |
| NOTICE crediting upstream | ✓ | Both — opentelemetry-proto for 3A; opentelemetry-proto + Apple swift-distributed-tracing for 3B. |
| Swift 6.0+ tools version | ✓ | Both. |
| macOS platform floor | mixed | swift-tracing-otlp at macOS 14; swift-distributed-tracing-bridge at macOS 15 (`Mutex` for `nonmutating`-set Span). Both valid per RFC-0003. |
| Sendable-clean by default | ✓ | All public types `Sendable`. Two `@unchecked Sendable` final classes (`OTLPTracer`, `OTLPSpan`) with documented Mutex justification. |
| Strict concurrency in CI | ✓ | Both. |
| Single public error enum, typed throws | ✓ | `TracingOTLPError`, `DistributedTracingBridgeError` (both empty in v0.1; reserved). |
| Public APIs Foundation-free | ✓ | None expose `Data`, `URL`, `Date`. swift-distributed-tracing has minimal transitive deps (only swift-service-context); preserves RFC-0001. |
| Repo skeleton from `bare-swift new` | ✓ | Both. |
| Tests/Vectors/EXCEPTIONS.md | ✓ | Both. |
| README tagline + ≤30-line example | ✓ | Both. |
| CHANGELOG with v0.1 entry | ✓ | Both. |
| CI green on macOS + Linux | ✓ | Both. |
| DocC bundle | ✓ | Both. |

### Deviations and findings

**1. swift-distributed-tracing-bridge required `docc-target` opt-in upfront.**
- Applied in Task 1 from the start (Phase 2 lesson absorbed). CI green on first try; no DocC failures from dep-side warnings.
- **Action:** none. This is now standard practice for any multi-dep package.

**2. Apple-adapter pattern crystallized over two packages.**
- swift-prometheus-metrics (Phase 2C) and swift-distributed-tracing-bridge (Phase 3B) share the same shape: final class with `Mutex` state, factory pattern, ServiceContext-or-equivalent for cross-call state, caller-driven flush.
- Pattern is unspoken but consistent. Worth codifying (RFC-0009 candidate above).
- **Action:** RFC if/when third Apple-adapter ships.

**3. Mutex non-copyability gotcha (Swift 6).**
- Capturing a Mutex-typed property as `let local = self.property; closure: { local.withLock { ... } }` fails at compile time with `'self.property' is borrowed and cannot be consumed`. Workaround: `[self]` capture in closure.
- Recorded as `feedback_swift_distributed_tracing` memory.
- **Action:** none beyond the memory. Watch for this in future Mutex-using packages.

**4. `nonmutating set` illegal on class properties.**
- Apple's Span protocol declares `var operationName: String { get nonmutating set }`. Class implementations satisfy via plain `set` — Swift's protocol witnessing handles it. Don't propagate the `nonmutating` keyword to class implementations (compile error).
- Recorded as memory.
- **Action:** none.

**5. ProtoWriter duplication between swift-otlp-exporter and swift-tracing-otlp.**
- ~250 LOC duplicated. Wire-format spec is stable; duplication risk is low. Tests in each package independently verify byte vectors; they cross-check by being byte-identical for shared inputs.
- **Action:** RFC-0010 candidate (low priority). Reconsider at logs-adapter ship if it ever happens.

### Verdict

RFC-0001 held cleanly under Phase 3 stress (the second Apple-frontend adapter; the first 4-dep package). The conventions absorb new patterns smoothly. No conventions wrong; the language has a few rough edges (Mutex non-copyability, `nonmutating` on class props) that recurring practice handles.

---

## Item 5 — Phase 4 anchor decision ✗ NOT YET (recommendation)

Phase 4 anchor candidates surveyed:

| Wave | Packages | Risk | Adoption fit |
|---|---|---|---|
| **Networking primitives** | swift-uri (WHATWG URL), swift-http-types-extras, swift-cookie, swift-mime | Medium | **High** — every server library needs URL/URI parsing, cookie handling, MIME types. Foundation-free alternatives genuinely scarce (swift-url is one option but heavy). |
| Config & serialization | swift-toml (TOML 1.0), swift-yaml (YAML 1.2), swift-msgpack, swift-cbor, swift-jsonpath | High | High — broad audience, but each package competes with established alternatives (Yams, swift-yaml, etc.). TOML/YAML are full-spec efforts (1–2 packages-worth each). |
| Crypto-adjacent | swift-blake3, swift-siphash, swift-hkdf | Medium-low | Medium — swift-crypto is established. Differentiation requires careful positioning (no Foundation, no CryptoKit, no Apple platform-specific code). Niche but real. |
| Logs (skipped Tranche 3C) | swift-log-otlp, swift-structured-logger | Low | Medium — swift-log already exists with mature adapters. Differentiation is OTLP log envelopes specifically (~300 LOC of work; not anchor-worthy). |

### Recommendation: **Anchor on the networking primitives wave**

Reasons:

1. **Audience continuity.** Phase 1's adopters (server-stack-adjacent) and Phase 2/3's adopters (observability) overlap heavily with networking primitive consumers. URL parsing in particular is the most-requested missing piece in the swift-server world.
2. **Foundation-free differentiation is real.** Foundation's URL is pre-WHATWG and has known quirks. swift-url exists but has heavy deps. A clean WHATWG-compliant URL parser with zero deps is genuinely valuable.
3. **Risk profile is known.** Each package is independently scoped and shippable; no cross-package coupling like the OTLP common-types question. Tranches can ship in any order.
4. **Builds on Phase 2 foundation.** swift-bytes (Phase 2A) is the right input/output type for these parsers; reuse is already idiomatic in the ecosystem.
5. **Sets up Phase 5 / config + serialization.** Once URL/URI/MIME exist, config/serialization parsers (TOML, YAML, MessagePack, CBOR) become more feasible — each can output via Bytes and consume via swift-bytes-aware readers.

Config/serialization is rejected as the anchor for the same reason it was rejected for Phase 2 and Phase 3: TOML/YAML are too big for a single anchor wave, and adoption-fit is "broad but each competes with established alternatives." Networking primitives are smaller, more focused, and have clearer differentiation.

Crypto-adjacent is rejected: swift-crypto is too entrenched, and the differentiation argument is harder to make.

Logs (the skipped 3C) is rejected: too small for an anchor; can ship as a Phase 4 side deliverable if requested.

### Phase 4 tranche sketch (subject to formal RFC)

| Tranche | Packages | Estimated calendar |
|---|---|---|
| 4A | swift-uri (WHATWG URL parser + serializer) — the headline package | ~1 day at observed pace; ~1–2 months at original budget |
| 4B | swift-mime (MIME parser + standard types), swift-cookie (RFC 6265 cookies) | ~1 day each |
| 4C | swift-http-types-extras (high-level helpers atop Apple's swift-http-types — first package depending on a *second* Apple package?) | ~1 day |
| 4D *(stretch)* | swift-log-otlp (the skipped Tranche 3C), or swift-jsonpath (continues format tier from Phase 1's swift-jsonpointer) | ~1 day if shipped |

Total Phase 4 budget: 2–6 weeks calendar at observed pace, or 4–6 months at original roadmap pace.

**Action:** author the anchor decision as **RFC-0008** (Phase 4 anchor) and accept it before Phase 4 plans start.

---

## Open work to clear Gate 3

1. **Write RFC-0008** (Phase 4 anchor: networking primitives). 1–2 hour task. References this retrospective.
2. *(Optional)* RFC-0009 (Apple-adapter pattern) and RFC-0010 (inter-package code-sharing) — not blocking; defer or skip.
3. **Begin Phase 4 planning** after RFC-0008 accepts.

**Recommended cadence:** RFC-0008 immediately after this retro is read → Phase 4 planning starts → first Tranche 4A package (swift-uri) within a session.

---

## What this retrospective changes for Phase 4

- The plan-then-execute-with-checkpoints workflow stays. Phase 3 demonstrated it scales to Apple-frontend adapter integration with three protocols (`Tracer`, `Span`, `Instrument`) and four upstream deps.
- Inline-vector test pattern stays.
- `docc-target` opt-in upfront for any multi-dep package (Phase 2 lesson, validated again in Phase 3).
- `Mutex` `[self]` capture pattern is now canonical for adapter packages.
- Cross-package symbol references via single-backtick stays.
- Apple-adapter shape (final class + factory + buffer + caller-driven flush) is the third reproducible pattern after StatsD/OTLP encoder and metrics-frontend adapter; ready to formalize if a third adapter ships (RFC-0009 candidate).
- Networking primitives (Phase 4) will pull `swift-bytes` heavily; ensure the existing `BytesReader` / `BytesWriter` API is usable for parsers (likely already is — swift-jsonpointer and swift-otlp-exporter both consume it well).

---

## Decision

Gate 3 is **provisionally PASSED on items 1, 2, 4** and **NOT YET PASSED on items 3 (partial) and 5 (open)**. Phase 4 plans should be drafted but not executed until item 5 closes via RFC-0008.

Phase 3 was the third tight, fast wave shipped on the AI-assisted execution model. Two packages shipped (one signal encoder, one Apple-frontend adapter) inside two days, with RFC-0007's anchor commitments fully met and the full OTel metrics+traces story complete. The CI process flushed real issues (Mutex non-copyability, `nonmutating set` on classes, SpanEvent init signature); no silent issues shipped. Conventions established at Gate 1 and reaffirmed at Gate 2 held under the second Apple-frontend adapter stress test.

The next step is **RFC-0008 anchoring Phase 4 on networking primitives**, which both clears Gate 3 and structures the next wave.

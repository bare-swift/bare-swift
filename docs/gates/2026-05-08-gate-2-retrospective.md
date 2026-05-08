# Gate 2 Retrospective: Phase 2 → Phase 3

**Date:** 2026-05-08
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 2 (the observability wave anchored by [RFC-0005](../../rfcs/0005-phase-2-anchor-observability.md)) against the criteria adapted from the roadmap's Phase 1 → 2 gate, and recommends whether Phase 3 should start.

The roadmap (§7) only formally specifies the Phase 1 → 2 gate. Gate 2's criteria mirror that structure, applied to Phase 2's anchored scope.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | Anchored Tranche 2A/2B/2C packages shipped at v0.1+, CI green, DocC live, listed | ✓ PASS |
| 2 | RFC-0005 commitments fulfilled (observability tier complete; Bytes prerequisite met) | ✓ PASS |
| 3 | ≥2 new RFCs accepted during Phase 2 (beyond founding + RFC-0005/0006) | partial — see below |
| 4 | RFC-0001 conventions stress-test against Phase 2 packages | ✓ DONE BELOW |
| 5 | Phase 3 anchor decision recorded as RFC | ✗ NOT YET (recommendation below) |

**Overall:** items 1, 2, 4 pass cleanly. Item 3 partially passes (no new RFCs were accepted *during* Phase 2 execution; the earlier RFC-0003/0004/0005/0006 cluster front-loaded the process work). Item 5 is open. Phase 3 plans should be drafted but not executed until item 5 closes.

The roadmap stop conditions did not trigger: Phase 2 calendar time was ~2 days (vs the 6–9 month budget — this matches Phase 1's pace and does not indicate "conventions or scope are wrong"); zero security incidents; no Apple announcements that subsumed planned packages.

---

## Item 1 — Phase 2 packages shipped ✓ PASS

| Tranche | Package | v0.1+ tag | DocC | Index | CI |
|---|---|---|---|---|---|
| 2A | swift-bytes | v0.1.0 | ✓ | ✓ | green |
| 2A | swift-hdrhistogram | v0.1.0 | ✓ | ✓ | green |
| 2B | swift-statsd | v0.1.0 | ✓ | ✓ | green |
| 2B | swift-otlp-exporter | v0.1.0 | ✓ | ✓ | green |
| 2C | swift-prometheus | v0.2.0 *(minor bump)* | ✓ | ✓ | green |
| 2C | swift-prometheus-metrics | v0.1.0 | ✓ | ✓ | green |

All shipped. The umbrella site at <https://bare-swift.github.io/bare-swift/> now lists 16 packages (10 from Phase 1, 5 new from Phase 2, plus swift-greet from Phase 0).

**Calendar time:** 2026-05-07 (swift-bytes) through 2026-05-08 (swift-prometheus-metrics) — roughly 2 days. This sits well inside the 6–9 month budget. The pace is suspicious-fast, but every package has full test coverage under strict concurrency, end-to-end consumer verification, DocC live, and was caught at every gate (CI required real fixes for swift-prometheus-metrics's DocC summary-line link and the swift-metrics name collision). The tests + plans + checkpoints workflow demonstrably caught bugs at each layer rather than letting them slip through.

---

## Item 2 — RFC-0005 commitments ✓ PASS

[RFC-0005](../../rfcs/0005-phase-2-anchor-observability.md) committed to the observability wave with Bytes as a prerequisite. Tracking against the RFC's Tranche sketch:

| RFC-0005 commitment | Outcome |
|---|---|
| Tranche 2A: swift-bytes (after RFC-0006) + swift-hdrhistogram | ✓ both shipped |
| Tranche 2B: swift-statsd + swift-otlp-exporter (depends on Bytes) | ✓ both shipped |
| Tranche 2C: swift-prometheus 0.2 (rebased on Bytes) + swift-prometheus-metrics | ✓ both shipped |
| RFC-0006 Bytes shipped before 2B (blocking dependency) | ✓ shipped first; 2B took both swift-bytes and (for OTLP) swift-varint |
| Convergence on `Bytes` output across observability tier | ✓ all three exporters (statsd / OTLP / prometheus) emit `Bytes` |
| Foundation-free guarantee (RFC-0001) | ✓ preserved across all packages, including the swift-metrics adapter |

Notable wins beyond the RFC's stated commitments:

- **swift-otlp-exporter scope expanded** from "metrics over HTTP+protobuf only" (planned) to all five OTLP metric types (Gauge, Sum, Histogram, ExponentialHistogram, Summary). The hand-rolled protobuf encoder built on swift-bytes + swift-varint validated the byte-buffer story in production-shaped wire format.
- **swift-prometheus-metrics adapter** completes the stack — any swift-metrics-instrumented codebase now gains Prometheus output via one bootstrap call, the integration point that makes the observability tier useful in real services.
- **First non-bare-swift dependency** (Apple's swift-metrics) added without violating RFC-0001 (swift-metrics has zero transitive deps).

The RFC's "open questions" resolved as expected:

| RFC-0005 open question | Resolution |
|---|---|
| Do we ship swift-prometheus-metrics in 2C or defer? | Shipped in 2C (closes the integration story). |
| OTLP scope: metrics-only or include traces/logs? | Metrics-only for v0.1, deferred per spec (right call — protobuf encoder for traces/logs is the same machinery; v0.2 is mechanical). |
| swift-statsd dialects: just DogStatsD or both? | Both Etsy and DogStatsD shipped. |
| swift-distributed-tracing integration | Deferred to Phase 3 as planned. |

---

## Item 3 — ≥2 new RFCs accepted during Phase 2 — partial

**Pre-Phase 2 (Gate 1 closeout):** RFC-0003 (platform floor), RFC-0004 (inline test vectors), RFC-0005 (Phase 2 anchor), RFC-0006 (Bytes buffer type) all accepted. These cleared Gate 1 item 3 retroactively and prepared Phase 2.

**During Phase 2 execution:** zero new RFCs accepted.

This is a partial result. The intent of the criterion (proves the RFC process is alive) is well-served by the four pre-Phase-2 RFCs, all of which were used by Phase 2 execution. But the spirit of "RFC process is real and ongoing" deserves at least one new RFC during Phase 2 to confirm.

**Candidates surfaced during Phase 2 execution that warrant RFCs:**

### RFC-0007 — Phase 3 anchor [the gating RFC for closing Gate 2]

Same shape as RFC-0005 was for Phase 2: pick the anchor wave, justify the choice, set the Tranche sketch. Decision space mapped in item 5 below.

### RFC-0008 — Cross-package DocC reference style [low priority]

**Trigger:** ``Bytes`` (DocC symbol-link) for swift-bytes types referenced from swift-statsd / swift-otlp-exporter / swift-prometheus / swift-prometheus-metrics fails under `--warnings-as-errors`. The convention "use single-backtick `` `Bytes` `` (code formatting) for cross-package types" is now a memory; should be codified as an addendum to RFC-0001 or its own short RFC.

### RFC-0009 — Inter-package dependency policy [medium priority]

**Trigger:** Phase 2 introduced the first inter-package deps in the ecosystem (swift-statsd → swift-bytes; swift-otlp-exporter → swift-bytes + swift-varint; swift-prometheus 0.2 → swift-bytes; swift-prometheus-metrics → swift-prometheus + swift-metrics). All worked, but no convention exists for: how do we choose between inlining a tiny utility vs taking an inter-package dep? When does a dep get pinned exactly vs ranged? What's the policy on transitive deps to non-bare-swift packages (swift-prometheus-metrics established that zero-transitive-dep is the bar, but it's not codified)?

### RFC-0010 — Pre-1.0 minor-bump breaking-change policy [low priority]

**Trigger:** swift-prometheus 0.1 → 0.2 changed the `exposition()` return type from `String` to `Bytes`. This is a breaking change in a minor bump. We followed the pre-1.0 SemVer convention (anything goes before 1.0) and used the `feat!:` Conventional Commit prefix, but the policy is implicit. Should be made explicit, especially as more packages bump beyond 0.1.

### Recommendation

Author and accept **RFC-0007** (Phase 3 anchor) as the next governance work. That alone clears the spirit of item 3.

RFC-0008/0009/0010 are low-stakes; either bundle into a single "Phase 2 lessons" RFC, defer to Phase 3, or skip — the patterns are already in memory and being applied.

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

Per-package review against RFC-0001 for the 6 Phase 2 packages (5 new + 1 minor bump):

### Compliance summary

| Convention | All 6 | Notes |
|---|---|---|
| Module name PascalCase, no `Swift` prefix | ✓ | `Bytes`, `HdrHistogram`, `StatsD`, `OTLPExporter`, `Prometheus`, `PrometheusMetrics` |
| Apache-2.0 + LLVM exception SPDX | ✓ | Every source file. |
| NOTICE crediting upstream | ✓ | All 6 (statsd → Etsy spec; OTLP → opentelemetry-proto; prometheus-metrics → swift-metrics + swift-prometheus). |
| Swift 6.0+ tools version | ✓ | All 6. |
| macOS platform floor | mixed | swift-bytes/hdrhistogram/statsd/otlp-exporter at macOS 14; swift-prometheus 0.2 + swift-prometheus-metrics at macOS 15 (inherited from `Synchronization`). Both are valid per RFC-0003. |
| Sendable-clean by default | ✓ | All public types `Sendable`. |
| Strict concurrency in CI | ✓ | All 6. |
| Single public error enum, typed throws | ✓ | `BytesError`, `HdrHistogramError`, `StatsDError`, `OTLPError` (empty in v0.1, reserved), no new error type for prometheus 0.2 or prometheus-metrics. |
| Public APIs Foundation-free | ✓ | None expose `Data`, `URL`, `Date`, etc. swift-prometheus-metrics depends on Apple's swift-metrics, which itself has zero transitive deps. |
| Repo skeleton from `bare-swift new` | ✓ | All 5 new packages. |
| Tests/Vectors/EXCEPTIONS.md | ✓ | All 5 new packages. |
| README tagline + ≤30-line example | ✓ | All 6. |
| CHANGELOG with v0.1 entry | ✓ | All 6 (prometheus 0.2 entry includes migration block per pre-1.0 minor-bump convention). |
| CI green on macOS + Linux | ✓ | All 6. |
| DocC bundle | ✓ | All 6. |

### Deviations and findings

**1. swift-prometheus-metrics CI required `docc-target` workaround.**
- DocC under `--warnings-as-errors` failed on dep-side (swift-metrics's) doc warnings when run without `--target`. Fix: bare-swift's reusable `package-ci.yml` now accepts a `docc-target` input; this package opts in.
- This is a precedent for any future multi-dep package. Recorded as memory.
- **Action:** none beyond the workflow change already shipped.

**2. swift-prometheus 0.2 introduced the first pre-1.0 breaking change.**
- `exposition()` return type changed from `String` to `Bytes`. SemVer pre-1.0 permits this; we used `feat!:` commit and a CHANGELOG migration block.
- The policy is implicit; RFC-0010 candidate above proposes codifying it.
- **Action:** RFC if/when a second pre-1.0 minor-bump landings makes the pattern matter.

**3. Cross-package DocC symbol references.**
- Multiple packages use single-backtick `` `Bytes` `` instead of double-backtick `` ``Bytes`` `` for swift-bytes types. DocC `--warnings-as-errors` treats unresolved cross-package symbol-links as errors. Convention is in memory but not codified.
- **Action:** RFC-0008 candidate (low priority).

**4. swift-metrics ↔ swift-prometheus type-name collision.**
- swift-metrics's `CoreMetrics.Counter`/`Gauge` user-facing structs collide with swift-prometheus's `Counter`/`Gauge` classes. Resolved via selective `import protocol CoreMetrics.CounterHandler` (and similar). Recorded as memory.
- This is a one-package issue (PrometheusMetrics is the only package depending on both); doesn't generalize. Doc note, not RFC.
- **Action:** none.

**5. Closure-literal typed-throws inference.**
- Closures passed as parameters with `throws(SpecificError)` types may have inner `try` fail to type-check ("invalid conversion of thrown error type 'any Error' to 'MetricsError'"). Workaround: drop closure parameter to plain `throws`. Recorded as memory.
- **Action:** none. Track upstream Swift bug; if fixed in 6.2+, retire.

**6. DocC summary line restriction.**
- The line right after `# ``Type`` ` is the summary; cannot contain Markdown links under `--warnings-as-errors`. Caught in CI for swift-prometheus-metrics; fixed by moving the link into Overview.
- **Action:** add to CONTRIBUTING.md as a footgun note (no RFC needed).

### Verdict

RFC-0001 held up cleanly under Phase 2 stress. Two new patterns crystallized (selective imports for type-collision, single-backtick DocC for cross-package refs); both are documented and applied. No conventions are wrong; the language and tooling have a few rough edges that recurring patterns absorb.

---

## Item 5 — Phase 3 anchor decision ✗ NOT YET (recommendation)

Phase 3 anchor candidates surveyed:

| Wave | Packages | Risk | Adoption fit |
|---|---|---|---|
| Distributed tracing | swift-tracing-otlp (OTLP/HTTP for traces; reuses swift-otlp-exporter's ProtoWriter), swift-distributed-tracing-bridge (Apple's frontend → bare-swift backend), swift-baggage-extras | Medium | High — completes the OpenTelemetry trio (metrics ✓ from Phase 2, tracing this wave, logs deferred). swift-otlp-exporter's ProtoWriter is already 80% of the work. |
| Logging | swift-log-prometheus (counter-per-level adapter), swift-log-otlp (OTLP log envelope encoder), swift-structured-logger (JSON/logfmt formatter) | Low-medium | Medium — swift-log already exists with good adapters; differentiation requires more thought. |
| Config & serialization | swift-toml, swift-yaml, swift-msgpack, swift-cbor, swift-jsonpath | High | High — broad audience, but each package competes with established alternatives; TOML/YAML are full-spec efforts (1–2 packages worth each). |
| Crypto-adjacent | swift-blake3, swift-siphash, swift-hkdf, swift-x25519-bridge | Medium | Medium — swift-crypto is the established option; bare-swift would need a clear differentiation story (no Foundation, no CryptoKit dep). |
| Networking primitives | swift-uri (WHATWG URL), swift-http-types-extras, swift-cookie, swift-mime | Medium | High — every server library needs these; current options are partial. |

### Recommendation: **Anchor on the distributed tracing wave**

Reasons:

1. **Completes OpenTelemetry coverage.** Phase 2's swift-otlp-exporter shipped metrics; tracing follows the same proto schema with different message types. The hand-rolled ProtoWriter is reusable. Logs can follow in Phase 4.
2. **Adoption continuity.** Same audience that adopted swift-otlp-exporter / swift-prometheus-metrics needs tracing next. Server stacks treating observability as a tier expect the trio (metrics + traces + logs).
3. **Risk profile is known.** swift-otlp-exporter validated the protobuf encoding approach; swift-tracing-otlp reuses 80% of the machinery. The adapter pattern from swift-prometheus-metrics carries over to swift-distributed-tracing.
4. **Bytes/Varint already ship.** No new foundation work required; the Phase 2 foundation pays off again.
5. **swift-distributed-tracing exists** as the Apple frontend (analogous to how swift-metrics existed for our Phase 2 work). Adapter pattern is proven.

Config & serialization is rejected as the anchor for the same reason RFC-0005 rejected it for Phase 2: TOML/YAML are full-spec efforts and adoption-fit is medium (each competes with existing options). Better fit for Phase 4.

Networking primitives is a strong candidate — defer to Phase 4 alongside or instead of config/serialization. Crypto is rejected (swift-crypto is too entrenched).

### Phase 3 tranche sketch (subject to formal RFC)

| Tranche | Packages | Estimated calendar |
|---|---|---|
| 3A | swift-tracing-otlp (OTLP exporter for traces; reuses ProtoWriter from swift-otlp-exporter) | ~1 day |
| 3B | swift-distributed-tracing-bridge (Apple frontend → bare-swift backend, mirrors swift-prometheus-metrics shape) | ~1 day |
| 3C | swift-baggage-extras OR swift-log-otlp (logs as OTLP envelopes) | ~1–2 days |

Total Phase 3 budget: ~1 week calendar time at observed Phase 2 pace, or 2–3 months at the original roadmap budget.

**Action:** author the anchor decision as **RFC-0007** (Phase 3 anchor) and accept it before Phase 3 plans start.

---

## Open work to clear Gate 2

To formally pass Gate 2 (clearing items 3 partially and 5):

1. **Write RFC-0007** (Phase 3 anchor: distributed tracing). 1–2 hour task. References this retrospective.
2. *(Optional)* Write RFC-0008 / RFC-0009 / RFC-0010 if desired — patterns are already absorbed into memory and CONTRIBUTING.md.
3. **Begin Phase 3 planning** after RFC-0007 accepts.

Item 1 is a single afternoon. Item 3 follows the established brainstorm → spec → plan → execute pattern.

**Recommended cadence:** RFC-0007 immediately after this retro is read → Phase 3 planning starts → first Tranche 3A package (swift-tracing-otlp) within a session.

---

## What this retrospective changes for Phase 3

- The plan-then-execute-with-checkpoints workflow stays. Phase 2 demonstrated it scales to mechanical-but-detailed work (hand-rolled protobuf encoder), focused refactors (swift-prometheus 0.2), and integration packages with non-bare-swift deps (swift-prometheus-metrics).
- Inline-vector test pattern stays (RFC-0004).
- Per-package CHANGELOGs and "Limitations / Out of scope for v0.1" sections both proved useful — keep doing them.
- Cross-package DocC reference style (single-backtick `` `Type` ``) is now table stakes for any package that depends on a bare-swift package. Add to CONTRIBUTING.md or RFC-0008.
- Closure-literal typed-throws gotcha is documented; future authors should default to plain `throws` on closure-typed parameters.
- Multi-dep packages should set `docc-target: <ModuleName>` in their CI workflow (the reusable workflow now supports this input).
- Pre-1.0 minor-bump breaking changes should use `feat!:` commit prefix + CHANGELOG migration block (swift-prometheus 0.2 set the precedent).

---

## Decision

Gate 2 is **provisionally PASSED on items 1, 2, 4** (technical and convention-compliance criteria) and **NOT YET PASSED on items 3 (partial) and 5 (open)**. Phase 3 plans should be drafted but not executed until item 5 closes via RFC-0007.

Phase 2 was a tighter, faster execution than Phase 1 — six packages shipped (and one minor-version bump that introduced the first pre-1.0 breaking change in the ecosystem) inside two days, with the RFC-0005 anchor commitments fully met and the observability tier converged on a single byte-buffer story. The CI process flushed real bugs (DocC summary-line links, cross-module type collision, dep-side DocC warnings); no silent issues shipped. The conventions established at Gate 1 held under integration-package stress (the first swift-metrics adapter), confirming RFC-0001 is robust.

The next step is **RFC-0007 anchoring Phase 3 on distributed tracing**, which both clears Gate 2 and structures the next wave of work.

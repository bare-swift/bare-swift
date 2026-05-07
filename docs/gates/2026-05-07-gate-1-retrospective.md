# Gate 1 Retrospective: Phase 1 → Phase 2

**Date:** 2026-05-07
**Status:** In progress
**Author:** bare-swift project lead

This document evaluates the formal Gate 1 criteria from the [roadmap](../../../docs/superpowers/specs/2026-05-04-bare-swift-roadmap-design.md) §7 and recommends whether Phase 2 should start.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All 10 packages shipped at v0.1+, CI green, DocC live, listed | ✓ PASS |
| 2 | Public launch landed; ≥1 external adopter | ✗ NOT YET |
| 3 | ≥2 RFCs accepted beyond founding two | ✗ NOT YET (candidates identified) |
| 4 | Conventions retrospective complete | ✓ DONE BELOW |
| 5 | Anchor decision recorded as RFC | ✗ NOT YET (recommendation below) |

**Overall:** Items 1 and 4 pass. Items 2, 3, 5 are open and block Phase 2 start. The shape of the path forward is clear: the technical work succeeded, the **launch and process work** still has to happen. None of the open items require new code; they're communications, RFCs, and a Phase 2 anchor decision.

The roadmap explicitly says "If items 1–4 fail, Phase 2 doesn't start; instead, re-plan Phase 1 finishing work." Item 4 passes; item 2 partially gates; item 3 partially gates. Recommendation: complete items 2, 3, 5 over the next 1–2 weeks before opening Phase 2 plans.

---

## Item 1 — All 10 packages shipped ✓ PASS

| Tranche | Package | v0.1.0 tag | DocC | Index | CI |
|---|---|---|---|---|---|
| 1A | swift-hex | ✓ | ✓ | ✓ | green |
| 1A | swift-base64 | ✓ | ✓ | ✓ | green |
| 1A | swift-crc | ✓ | ✓ | ✓ | green |
| 1B | swift-uuid | ✓ | ✓ | ✓ | green |
| 1B | swift-varint | ✓ | ✓ | ✓ | green |
| 1B | swift-xxhash | ✓ | ✓ | ✓ | green |
| 1B | swift-jsonpointer | ✓ | ✓ | ✓ | green |
| 1C | swift-dotenv | ✓ | ✓ | ✓ | green |
| 1C | swift-ddsketch | ✓ | ✓ | ✓ | green |
| 1C | swift-prometheus | ✓ | ✓ | ✓ | green |

All shipped. The umbrella site at https://bare-swift.github.io/bare-swift/ lists all 10 packages plus the swift-greet demo (11 total). The Phase 0 → Phase 1 acceptance gate (full pipeline working end-to-end) was validated on every package.

**Calendar time:** Tranche 1A (hex, base64, crc) through Tranche 1C (prometheus) shipped within the same calendar week of 2026-05. This is well inside the 4–6 month budget the roadmap allotted. The roadmap's Stop Condition "Phase 1 takes >9 months → conventions are wrong" did not trigger.

---

## Item 2 — Public launch ✗ NOT YET

The roadmap requires:

> Public launch landed (blog post, Swift Forums announcement, HN/Lobsters post). Adoption signal collected: ≥1 external user across the ecosystem (cumulative, not per-package).

Status: **none of the launch artifacts have been authored or posted.**

### What's needed

1. **Blog post / announcement page on the umbrella site.**
   - Frame: "10 small, Foundation-free Swift packages, with relative-error guarantees on the heavy ones."
   - Concrete code samples from at least one package per tier (encoding/hashing/foundation/format/config/observability).
   - Link to each package's DocC.
   - Mention the bare-swift CLI as the on-ramp for contributors.
2. **Swift Forums announcement** in `Show & Tell` (or `Server`).
3. **HN / Lobsters** post with the umbrella link.
4. **External adoption tracking.** Even one shipping app, library, or tutorial that pulls a bare-swift package counts.

This is a 2–4 hour writing exercise plus the wait for adoption signal. None of it is blocked by missing code.

### Risk

The roadmap's stop condition: *"Zero external adoption after Tranche 1C public launch → packaging/positioning failed; rethink before Phase 2."* We literally cannot measure adoption without launching first. The launch is the prerequisite for the measurement.

---

## Item 3 — ≥2 new RFCs ✗ NOT YET (candidates below)

Currently accepted: RFC-0001 (API conventions), RFC-0002 (test parity policy). Both are founding RFCs from Phase 0.

The Gate requires **at least 2 RFCs accepted beyond** those two — to prove the RFC process is real, used, and produces accepted decisions, not just exists on paper.

Candidates surfaced by Phase 1 execution:

### RFC-0003 — Platform floor & Synchronization primitives [strong candidate]

**Trigger:** swift-prometheus required `import Synchronization` (`Atomic`, `Mutex`), which raises the macOS platform floor from 14 to 15.

**Question:** does the org-wide floor stay at macOS 14 with per-package overrides for libraries that need newer primitives, or do we bump the floor uniformly to macOS 15 to align with Swift 6.0+ defaults?

Decision is real and impacts every Phase 2 package. Worth a formal RFC.

### RFC-0004 — Test vector strategy: inline literals [strong candidate]

**Trigger:** Every Phase 1 package adopted the same pattern: `Tests/Vectors/EXCEPTIONS.md` documents the upstream Rust crate's test approach; vectors are baked into Swift literals (no runtime JSON / fixture loading). RFC-0002 calls for "extraction" but doesn't specify mechanism; we converged on "extract once at plan time, bake inline".

**Proposal:** codify the inline-literals + EXCEPTIONS.md pattern as the canonical Phase 2 practice. Optionally clarify when external generators (like the Python `ddsketch` reference for swift-ddsketch) are acceptable as vector sources.

### RFC-0005 — `@unchecked Sendable` justification standard [medium candidate]

**Trigger:** swift-prometheus has 5+ `@unchecked Sendable` types. RFC-0001 requires "an inline comment justifying the unchecked claim". A doc-comment-only justification (as we used) is technically off-spec.

**Proposal:** clarify whether doc-comment-level or inline-comment-level justification is required, and standardize the justification template.

### RFC-0006 — Bytes / span-like buffer type [strong candidate, blocks observability wave]

**Trigger:** Phase 1 packages all use `[UInt8]` or `some Sequence<UInt8>` as the byte type. This is fine for hashing/encoding, but Phase 2's OTLP-exporter and serialization waves will demand a more efficient buffer type that doesn't copy. The roadmap §4 already flags Bytes/ByteBuffer as needing an RFC.

**Proposal:** RFC defining the bare-swift byte-buffer story — adopt swift-nio's `ByteBuffer` (and accept the dependency), wait for stdlib `RawSpan` / `Span` to stabilize, or roll our own. Each option has costs.

### Recommendation

Author and accept **RFC-0003** (platform floor) and **RFC-0004** (test vector strategy) immediately — both are short, both reflect already-proven patterns, both are unblocked. That clears Gate item 3.

RFC-0006 (Bytes) should be the **first RFC of Phase 2**, blocking the OTLP exporter and any serialization work. RFC-0005 is amendable later.

---

## Item 4 — Conventions retrospective ✓ DONE

Per-package review against RFC-0001:

### Compliance summary

| Convention | All 10 | Notes |
|---|---|---|
| Module name PascalCase, no `Swift` prefix | ✓ | `Hex`, `Base64`, `CRC`, `UUID`, `Varint`, `XXHash`, `JSONPointer`, `DotEnv`, `DDSketch`, `Prometheus` |
| Apache-2.0 + LLVM exception SPDX | ✓ | Every source file. |
| NOTICE crediting upstream Rust crate | ✓ | All 10. |
| Swift 6.0+ tools version | ✓ | All 10. |
| macOS 14+ platform | 9 / 10 | **swift-prometheus requires macOS 15** — see deviation below. |
| Sendable-clean by default | ✓ | All public types are `Sendable` (some `@unchecked`). |
| `@unchecked Sendable` justified | partial | Some justifications are doc-comment-level rather than inline-comment-level — see deviation. |
| Strict concurrency in CI | ✓ | All 10. |
| Single public error enum, typed throws | ✓ | `HexError`, `Base64Error`, etc. — typed throws on all throwing public functions. |
| Pure-logic packages stay sync | ✓ | No package added `async` for symmetry. |
| Public APIs Foundation-free | ✓ | None expose `Data`, `URL`, `Date`, etc. |
| Repo skeleton from `bare-swift new` | ✓ | All 10. |
| Tests/Vectors/EXCEPTIONS.md | ✓ | All 10. |
| README tagline + ≤30-line example | ✓ | All 10. |
| CHANGELOG with v0.1 entry | ✓ | All 10. |
| CI green on macOS + Linux | ✓ | All 10. |
| DocC bundle | ✓ | All 10. |

### Deviations and findings

**1. swift-prometheus platform floor: macOS 15 (RFC-0001 says 14+).**
- Cause: `Synchronization.Atomic` and `Synchronization.Mutex` are macOS 15 / Swift 6.0+ primitives.
- Was the right call given the package needs lock-free atomic counters; alternatives (NSLock, swift-atomics dependency, full mutex) all worse for hot-path metrics.
- **Action:** RFC-0003 (above) decides whether this becomes the org-wide floor or stays per-package.

**2. `@unchecked Sendable` justification placement.**
- swift-prometheus has 5 reference types each annotated `@unchecked Sendable`. The justification ("internal atomic / mutex guards correctness") is in the doc comment for the type, not as an inline comment on the `@unchecked` annotation itself.
- RFC-0001 wording is ambiguous on placement.
- **Action:** RFC-0005 clarifies.

**3. `@usableFromInline` propagation pitfall.**
- During swift-ddsketch execution, `@usableFromInline` was applied preemptively on internal types. This required propagating the annotation through the whole reference graph and broke when an internal type referenced another internal type. Removed all `@usableFromInline` from v0.1; performance impact is zero in debug, negligible in release.
- **Action:** documentation note, not RFC-worthy.

**4. Swift 6.1 Linux type-checker timeout on chained arithmetic.**
- swift-xxhash hit a timeout on a 4-way `&+` rotation merge (`((v1 << 1) | (v1 >> 31)) &+ ((v2 << 7) | (v2 >> 25)) ...`). Splitting into typed locals fixed it. Now used as a precaution in every package since.
- **Action:** add a code-style note to `CONTRIBUTING.md` (no RFC needed). Track upstream Swift bug; if fixed in 6.2+, retire the workaround.

**5. Foundation in tests for `log` / `pow` / `sqrt`.**
- swift-ddsketch's IndexMappingTests imports Foundation for `log`/`pow` round-trip math; Sources/ uses `#if canImport(Darwin) / Glibc / Musl`. Public API is Foundation-free; tests are pragmatic.
- RFC-0001 explicitly permits Foundation in tests ("internal use of Foundation is permitted where pragmatic"). Compliant.
- **Action:** none.

**6. Plan-execution corrections.**
- Every plan ran into at least one deviation from the as-written plan. Recurring patterns:
  - **Algorithmic spec details** (XXH3 mixers, DDSketch quantile shortcut) needed correction against the C/Rust reference at execution time.
  - **Test expectations** (uniform-quantile linear interpolation vs rank-value) needed correction against actual algorithm behavior.
  - **Compiler quirks** (non-copyable Atomics, ambiguous initializer overloads, DocC link disambiguators) surfaced only at compile time.
- The TDD pattern (write failing test → run → fix) caught all of these. **No silent bugs shipped.**
- **Action:** none required. The plan-then-execute-with-tests workflow is validated.

### Verdict

RFC-0001 conventions held up under Phase 1 stress. Two real deviations (platform floor, `@unchecked` placement) — both resolvable by RFC amendments rather than convention rewrites. The conventions are not wrong; they need clarification.

---

## Item 5 — Anchor decision for Phase 2 ✗ NOT YET (recommendation)

The roadmap §7 lists three candidate Phase 2 waves:

| Wave | Packages | Risk | Adoption fit |
|---|---|---|---|
| Config & serialization | swift-toml, swift-yaml, swift-msgpack, swift-cbor, swift-jsonpath, swift-bytes (after RFC) | High — TOML/YAML are big; Bytes needs RFC first | Medium — broad audience, but each package competes with established alternatives |
| Observability | swift-hdrhistogram, swift-otlp-exporter, swift-statsd, expanded swift-prometheus | Medium — known territory after ddsketch + prometheus | High — Phase 1 already shipped two adoption-magnet observability packages (ddsketch, prometheus); Phase 2 audience is the same |
| Routing/middleware | swift-router (matchit-equivalent), swift-tower, swift-mime, swift-cookie | Medium-high — middleware composition is a real design problem | Medium — server framework audience, more fragmented |

### Recommendation: **Anchor on the observability wave**

Reasons:

1. **Adoption continuity.** Phase 1's ddsketch + prometheus targeted exactly the audience this wave continues to serve. Whoever adopts swift-prometheus in Phase 1 is the natural candidate for swift-otlp-exporter or swift-statsd in Phase 2.
2. **Risk profile.** We just demonstrated competence with quantile sketches and metric atomics. HDRHistogram and StatsD reuse the same shape (atomic counters, format encoders); OTLP is more involved but well-specified.
3. **Bytes RFC unblocks one wave.** OTLP's protobuf encoding will demand a real byte-buffer story (RFC-0006). Approving that RFC during Phase 2 prep also prepares the Phase 3 config/serialization wave.
4. **Phase 1's Tranche 1C goal.** The roadmap explicitly described Tranche 1C as the adoption-magnet tranche; building on its momentum is the obvious move.

Config/serialization remains a strong Phase 3 candidate. Routing/middleware is intentionally deferred — designing tower-equivalent middleware composition without first answering the byte-buffer question is putting the cart before the horse.

### Phase 2 tranche sketch (subject to formal RFC)

| Tranche | Packages | Estimated duration |
|---|---|---|
| 2A | swift-bytes (after RFC), swift-hdrhistogram | ~6 weeks |
| 2B | swift-statsd, swift-otlp-exporter (depends on 2A's bytes) | ~6 weeks |
| 2C | swift-prometheus 0.2 (Summary type, native histograms, OpenMetrics) + swift-prometheus-metrics (swift-metrics adapter) | ~4 weeks |

Total Phase 2 budget: ~4 months calendar, mirroring Phase 1's pace.

**Action:** author this anchor decision as **RFC-0007** (or similar) and accept it before Phase 2 plans start.

---

## Open work to clear Gate 1

To formally pass Gate 1 (clearing items 2, 3, 5):

1. **Write RFC-0003** (platform floor & Synchronization). 1–2 hour task. Adopts already-shipped reality.
2. **Write RFC-0004** (test vectors & EXCEPTIONS.md). 1–2 hour task. Codifies already-shipped reality.
3. **Write Phase 2 anchor RFC** (observability wave + Bytes prerequisite). 2–3 hour task. References this retrospective.
4. **Write the launch announcement.** 2–4 hour task. Existing 10 packages provide concrete examples.
5. **Post the launch.** Swift Forums + HN/Lobsters; reply-friendly.
6. **Wait for adoption signal.** Even one external user clears item 2.

Items 1–3 can ship in a single afternoon of writing. Items 4–5 are a separate session focused on copy. Item 6 requires waiting (probably a week after the launch).

**Recommended cadence:** RFC writing block this week → launch next week → 1-week observation window → Gate 1 closeout review → Phase 2 planning.

---

## What this retrospective changes for Phase 2

- The 4 RFC candidates above (especially Bytes) become Phase 2 prep work.
- The CLI scaffolder works well enough that it shouldn't change. Skip "scaffold v2" temptations.
- The plan-then-execute-with-checkpoints workflow stays. Each Phase 2 package will follow the same pattern observed in Phase 1: spec → plan → inline execution with checkpoints, with the test layer catching plan-level errors.
- Inline-vector test pattern stays.
- Per-package CHANGELOGs and explicit "Limitations / Out of scope for v0.1" sections both proved useful — keep doing them.
- Linux Swift 6.1 type-checker workaround (typed locals for chained arithmetic) is now table stakes; mention in CONTRIBUTING.md.

---

## Decision

Gate 1 is **provisionally PASSED on the technical criteria** (items 1, 4) and **NOT YET PASSED on the process / launch / RFC criteria** (items 2, 3, 5). Phase 2 plans should be written but not executed until items 2, 3, 5 close.

# Gate 6 Retrospective: Phase 6 → Phase 7

**Date:** 2026-05-10
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 6 (the JSON tier wave anchored by [RFC-0011](../../rfcs/0011-phase-6-anchor-json-tier.md)) against the Gate 1–5 criteria template and recommends whether Phase 7 should start. All four tranches (6A–6D) shipped — third consecutive phase to ship its stretch (4D, 5D, 6D). The v0.2 downstream sweep committed in RFC-0011 also completed in parallel: six packages bumped to wire `Time` types in additively.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All anchored Tranche 6A/6B/6C/6D packages shipped at v0.1+, CI green, DocC live, listed | ✓ PASS |
| 2 | RFC-0011 commitments fulfilled (JSON tier + downstream v0.2 Time sweep) | ✓ PASS |
| 3 | ≥1 new RFC accepted during Phase 6 | ✗ NONE — see below |
| 4 | RFC-0001 conventions stress-test against Phase 6 packages | ✓ DONE BELOW |
| 5 | Phase 7 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0012 below) |

**Overall:** Items 1, 2, 4 pass cleanly. Item 3 fails (no in-flight RFCs). Item 5 is open. Phase 7 plans should be drafted but not executed until item 5 closes via RFC-0012.

**Item 3 is the first hard miss in five gates.** Gates 1/2/4/5 each had at least one RFC during execution; Phase 6 had none. The reason is straightforward: every cross-cutting question that arose was already covered by RFC-0010 (date/time policy) or by the established memory rules (DocC overload disambiguators, GitHub Pages setup). Operationally the absence is fine — there were no policy gaps to close mid-flight. But the criterion as stated wants to see the RFC machinery exercise; Phase 6 didn't trigger any.

The roadmap's stop conditions did not trigger: Phase 6 calendar time was ~1 day for the four new tranches plus ~1 hour for the v0.2 sweep across six packages. No security incidents. No Apple announcements that subsumed planned packages.

---

## Item 1 — Phase 6 packages shipped ✓ PASS

| Tranche | Package | v0.1+ tag | Tests | DocC | Index | CI |
|---|---|---|---|---|---|---|
| 6A | swift-json | v0.1.0 | 48 | ✓ | ✓ | green |
| 6B | swift-jsonpath | v0.1.0 | 50 | ✓ | ✓ | green |
| 6C | swift-jsonpatch | v0.1.0 | 40 | ✓ | ✓ | green |
| 6D | swift-jsonschema-lite | v0.1.0 | 25 | ✓ | ✓ | green |

All four tranches shipped. The 6D stretch (swift-jsonschema-lite) was selected per RFC-0011's offered choice between schema-lite, continued v0.2 sweep, and a yaml-lite alternative — schema-lite paired naturally with the rest of the JSON tier and closed RFC-0011's offered options cleanly.

Plus the **downstream v0.2 sweep** committed in RFC-0011 completed in the same session:

| Package | v0.2.0 addition | Tests added |
|---|---|---|
| swift-cookie | `Cookie.expiresAt` / `.expiresInstant` getters; `init(...expiresAt:)` factory | 6 |
| swift-msgpack | `MsgPackValue.timestamp(_:)` factory + `.asTimestamp` getter (timestamp32/64/96 wire forms) | 10 |
| swift-cbor | `CBORValue.dateString(_:)` (tag 0) + `.dateEpoch(_:)` (tag 1) + `.asDate` | 12 |
| swift-toml | `TOMLValue.asCalendar` / `.asInstant` / `datetime(from:)` | 8 |
| swift-tracing-otlp | `OTLP.Span.init(...startTime:endTime:)` + `.startTime` / `.endTime` / `.time` (event) | 5 |
| swift-log-otlp | `OTLP.LogRecord.init(time:observedTime:)` + `.time` / `.observedTime` | 4 |

**45 new tests across the v0.2 sweep**; **163 new tests across the four 6A–6D packages**; **208 new tests in Phase 6 total**.

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists 30 packages.

**Calendar time:** Phase 6 executed in ~1 day (4 new packages + 6 minor releases). swift-jsonpath was the densest single package this phase (~430 LOC parser + ~280 LOC evaluator with full RFC 9535 grammar including filter expressions and four function extensions).

---

## Item 2 — RFC-0011 commitments ✓ PASS

[RFC-0011](../../rfcs/0011-phase-6-anchor-json-tier.md) committed to the JSON tier wave plus the parallel v0.2 sweep:

| RFC-0011 commitment | Outcome |
|---|---|
| Tranche 6A: swift-json (RFC 8259 parser + serializer) | ✓ shipped |
| Tranche 6B: swift-jsonpath (RFC 9535 path queries) | ✓ shipped |
| Tranche 6C: swift-jsonpatch (RFC 6902 patch operations) | ✓ shipped |
| Tranche 6D *(stretch)*: pick from swift-jsonschema-lite / continued v0.2 sweep / swift-yaml-lite | ✓ shipped — picked swift-jsonschema-lite |
| **Parallel work-stream**: downstream v0.2 sweep wiring `Time` into 6 packages | ✓ all 6 shipped same session |
| Foundation-free, non-Codable across the wave | ✓ preserved across all packages |
| `JSONValue` integer/double split for precision fidelity | ✓ — used by `JSONValue`, propagated to `enum`/`const` numeric equivalence in jsonschema-lite |

The RFC's "out of scope" list (full IRegexp validation, custom function extensions, JSON Merge Patch, full JSON Schema 2020-12) held in every package's v0.1 release.

Notable findings beyond the RFC's stated commitments:

- **First two-dep package in the format tier**: swift-jsonpatch and swift-jsonschema-lite both depend on swift-json + swift-jsonpointer. Phase 1's swift-jsonpointer was authored "JSON-tree agnostic" specifically so something like JSONValue could be added later — that authoring decision held; jsonpointer needs no v0.2 to support the new consumers.
- **Negative-Instant clamping pattern** documented across three OTLP packages (tracing/log/exporter via the v0.2 sweep). `UInt64` proto3 wire fields can't represent negative nanos; bit-pattern encoding would silently corrupt downstream collectors. Established now as the canonical pattern for any Time → proto3 conversion.
- **DocC overload-disambiguator trap caught twice** (swift-jsonpatch v0.1.0 and swift-jsonschema-lite v0.1.0). The `-some_overload` suffix DocC suggests for ambiguous symbol references is fragile — fails to resolve even within the same module. Workaround: prose-form single-backtick references. Memory rule extends from cross-package to same-module overloads.

---

## Item 3 — ≥1 new RFC accepted during Phase 6 ✗ FAIL

**Pre-Phase 6 (Gate 5 closeout):** RFC-0011 (Phase 6 anchor) accepted 2026-05-10 to clear Gate 5 item 5.

**During Phase 6 execution:** **zero RFCs accepted.**

This is the first hard miss on item 3 across five gates. Gates 1/2/4/5 each had at least one in-flight RFC; Gate 3 had a partial pass (anchor only).

**Why:** every cross-cutting question that arose was already covered by RFC-0010 (date/time policy) or by established memory feedback (DocC rules, GitHub Pages setup). Phase 6 was the most-scaffolded phase of the project — RFC-0001 conventions were stable, the format-tier pattern was well-established, and the v0.2-sweep migration plan from RFC-0010 was already written. There were no policy gaps to close mid-flight.

Operationally this is fine — RFCs forced into existence to satisfy the criterion would be ceremony, not value. But the criterion as written wants to see the RFC machinery exercise.

**Candidates surfaced during Phase 6 execution that warrant RFCs (next gate's work):**

### RFC-0012 — Phase 7 anchor [the gating RFC for closing Gate 6]

Same shape as RFC-0005 / 0007 / 0008 / 0009 / 0011. Decision space mapped in item 5 below.

### RFC-0013 *(deferred)* — DocC linking conventions (cross-package + same-module)

**Trigger:** The `-some_overload` disambiguator failed in two Phase 6 packages (jsonpatch, jsonschema-lite). The `feedback_docc_cross_package` memory covers cross-package only; the trap also applies to same-module symbol references with overloads.

**Proposal:** Codify the DocC convention. Use single-backtick prose for: (a) cross-package symbol references, (b) overloaded same-module symbol references where the DocC disambiguator format is fragile, (c) topics-list items that would fail "non-link item" validation.

**Status:** memory rule already covers cases (a) and informally (b). Promoting to an RFC is low-priority; defer until a third trigger appears.

### Recommendation

Author and accept **RFC-0012** as the Phase 7 anchor. Skip RFC-0013 — the memory rule is operationally sufficient.

The item-3 miss should be acknowledged but not "fixed" — forcing RFCs to satisfy the criterion would be ceremony. Future gates may reframe item 3 as "≥1 *substantive* RFC accepted" to allow gates with no policy gaps to pass cleanly.

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

Per-package review against RFC-0001 for the 4 Phase 6 packages and the 6 v0.2 sweep packages:

| Convention | All ten | Notes |
|---|---|---|
| Module name PascalCase, no `Swift` prefix | ✓ | `JSON`, `JSONPath`, `JSONPatch`, `JSONSchemaLite` (the four new); v0.2 packages preserve module names. |
| Apache-2.0 + LLVM exception SPDX | ✓ | Every source file. |
| NOTICE crediting upstream | ✓ | All four credit IETF specs (RFC 8259, RFC 9535, RFC 6902, JSON Schema 2020-12). |
| Swift 6.0+ tools version | ✓ | All ten. |
| macOS platform floor | ✓ | All ten at macOS 14. |
| Sendable-clean by default | ✓ | All public types `Sendable`. No `@unchecked` needed. |
| Strict concurrency in CI | ✓ | All ten. |
| Single public error enum, typed throws | ✓ | `JSONError` (7), `JSONPathError` (9), `JSONPatchError` (8), `JSONSchemaError` (1). |
| Public APIs Foundation-free | ✓ | None expose Foundation types. JSONPath uses Swift stdlib `Regex` (not Foundation). |
| Repo skeleton from `bare-swift new` | ✓ | All four new packages. |
| Tests/Vectors/EXCEPTIONS.md | ✓ | All four new. |
| README tagline + ≤30-line example | ✓ | All four new. |
| CHANGELOG with v0.1 entry (or v0.2 for sweep) | ✓ | All ten. |
| CI green on macOS + Linux | ✓ | All ten. |
| DocC bundle | ✓ | All four new. |

### Deviations and findings

**1. JSON `JSONValue.integer` / `.double` split is the precision-fidelity pattern that propagates.**
- swift-json's split mirrors swift-msgpack's `int(Int64)` / `uint(UInt64)` and swift-cbor's `uint` / `negative(UInt64)`. The pattern is now ecosystem-canonical for any value type representing a JSON-shaped tree.
- swift-jsonschema-lite honours numeric equivalence in `enum`/`const` per JSON Schema spec — `.integer(1)` and `.double(1.0)` compare equal there but not at the value-type level.
- **Action:** none. Pattern is established and consistently applied.

**2. JSONPath's stdlib-`Regex`-superset-of-IRegexp choice documented.**
- RFC 9535 mandates I-Regexp (a subset of XSD-style). Swift stdlib's `Regex` is more permissive (accepts I-Regexp patterns and more). v0.1 ships with stdlib `Regex`; strict-I-Regexp validation is a v0.2 hardening item.
- **Action:** none beyond the doc note. v0.2 may add validator if a real consumer hits an interop case.

**3. DocC overload-disambiguator (`-some_overload`) is fragile within the same module.**
- Caught in jsonpatch and jsonschema-lite v0.1.0. DocC's symbol-reference syntax can't resolve `init(_:)-some_overload` even when there's exactly one matching init.
- Workaround: write the API name in single-backtick prose form. Same rule that applied to cross-package references now applies to same-module overloads.
- **Action:** memory rule covers cross-package; RFC-0013 candidate documents the broader pattern but is low-priority. Watch for a third trigger.

**4. Two-dep format-tier packages (jsonpatch, jsonschema-lite) added cleanly.**
- swift-json + swift-jsonpointer combined work fine; no version-pin conflicts because both are at v0.1.0 from a stable point in time.
- **Action:** none. Demonstrates the dependency-pinning convention scales.

**5. v0.2 migration sweep verifies "additive only" works in practice.**
- Six packages bumped, no source-breaking changes. Each migration was 30-60 minutes of work. Pattern is now proven repeatable.
- **Action:** none. Future foundation-tier additions (e.g. a hypothetical bignum or SIMD type) can follow the same migration cadence.

### Verdict

RFC-0001 held cleanly under Phase 6 stress (the largest tier wave to date — 4 new packages + 6 minor releases in one session). Conventions absorbed new patterns smoothly. The format tier is now the largest in the ecosystem with 9 packages (jsonpointer, dotenv, mime, cookie, uri, msgpack, cbor, toml, json) plus the four JSON-tier additions = 13 total.

---

## Item 5 — Phase 7 anchor decision ✗ NOT YET (recommendation)

Phase 7 anchor candidates surveyed:

| Wave | Packages | Risk | Adoption fit |
|---|---|---|---|
| **HTTP body codecs / compression** | swift-deflate, swift-gzip, swift-zlib; swift-content-encoding or swift-brotli as 7D | Medium-high (DEFLATE/INFLATE is genuinely substantial) | **High** — every HTTP server / client deals with `Content-Encoding: gzip`. Foundation-free decompression is a real gap. |
| Crypto-adjacent | swift-blake3, swift-siphash, swift-hkdf, swift-x25519 | Medium-low | Medium — third rejection in Phases 4/5/6. swift-crypto remains too entrenched. |
| Internationalization | swift-idna, swift-publicsuffix | Medium-high (IDNA needs Unicode tables; PSL is data-heavy) | Medium — closes swift-uri's IDNA deferral and swift-cookie's domain-validation gap. Niche but real. |
| OTLP/json variants | swift-otlp-json (cross-signal), swift-w3c-tracecontext, swift-otlp-resource | Low-medium | Medium — closes Phase 2/3/4 JSON-mode deferral. Same audience as existing OTLP packages. |
| HTTP-types-extras | swift-multipart, swift-range, header-validation extras | Low-medium | Medium — server-stack adopters always hit multipart eventually. |

### Recommendation: **Anchor on the HTTP body codecs / compression wave**

Reasons:

1. **Largest unfilled gap.** Every HTTP server and client needs to decompress incoming `Content-Encoding: gzip` request bodies and encode outgoing responses. Foundation-free options are genuinely missing in the Swift ecosystem; `Compression.framework` is Apple-platform-only and Foundation-bound.
2. **Audience continuity is total.** Phase 4's networking primitives (uri, mime, cookie) and Phase 6's JSON tier consumers all eventually serve compressed HTTP bodies.
3. **Builds on Phase 2 foundation.** swift-bytes is the natural input/output type for any byte-stream compressor.
4. **Differentiation is real.** A clean Foundation-free DEFLATE/INFLATE implementation with no `Compression.framework` / `zlib.h` C-binding is genuinely missing.
5. **Risk profile is known.** DEFLATE/INFLATE is moderately complex (Huffman tables, sliding window) but the spec (RFC 1951) is well-bounded. The natural shipping order is **decompression first** (INFLATE: ~600 LOC), with compression (DEFLATE: ~1500 LOC) as v0.2 — same staging pattern as swift-uri's "v0.1 ASCII-fast-path IDNA, v0.2 full UTS46".
6. **Sets up Phase 8+.** Once compression exists: OTLP/json with gzip-encoded payloads becomes feasible; `Accept-Encoding` content negotiation in HTTP frameworks becomes one-line; static-file servers can serve pre-compressed assets.

### Phase 7 tranche sketch (subject to formal RFC)

| Tranche | Packages | Estimated calendar |
|---|---|---|
| 7A | swift-deflate (RFC 1951 INFLATE; DEFLATE deferred to v0.2) | ~1–2 days at observed pace |
| 7B | swift-gzip (RFC 1952 wrapper around deflate) | ~1 day |
| 7C | swift-zlib (RFC 1950 wrapper; adds adler32 framing) | ~0.5 day |
| 7D *(stretch)* | swift-content-encoding (HTTP `Content-Encoding` header multiplexer) OR swift-brotli (RFC 7932) | ~1–2 days |

The "INFLATE-only in v0.1" decision is the key staging choice. Decompression is the dominant use case (servers receiving compressed requests, clients receiving compressed responses), and compression is genuinely harder. Half the value at one-third the effort. The v0.2 sweep adds DEFLATE encoding once the decoder is stable.

Total Phase 7 budget: 1–2 weeks calendar at observed pace; 4–6 months at original roadmap pace.

### Why other waves were rejected

- **Crypto-adjacent** — fourth consecutive rejection. The differentiation argument hasn't gotten stronger; swift-crypto is still the entrenched answer.
- **Internationalization** — niche audience compared to compression. swift-uri's IDNA gap is real but small. Better as a Phase 8+ side deliverable when the audience asks.
- **OTLP/json variants** — fits a documented deferral but the audience overlap with HTTP body codecs is total. If we ship compression first, OTLP/json + gzip together becomes a one-line Phase 8 wave.
- **HTTP-types-extras** — too narrow to anchor; better as a side deliverable.

**Action:** author the anchor decision as **RFC-0012** (Phase 7 anchor) and accept it before Phase 7 plans start.

---

## Open work to clear Gate 6

1. **Write RFC-0012** (Phase 7 anchor: HTTP body codecs / compression). 1–2 hour task. References this retrospective.
2. **Begin Phase 7 planning** after RFC-0012 accepts.

**Recommended cadence:** RFC-0012 immediately after this retro is read → Phase 7 planning starts → first Tranche 7A package (swift-deflate INFLATE) within a session.

---

## What this retrospective changes for Phase 7

- The plan-then-execute-with-checkpoints workflow stays.
- "Spec → plan → checkpointed inline execution" continues to scale.
- Inline-vector test pattern stays; RFC 6902 Appendix-A vectors validated swift-jsonpatch.
- `docc-target` opt-in upfront stays canonical.
- GitHub Pages two-step setup is now standard runbook for new repos. Six new repos in Phases 4–6 hit the same one-step fix.
- Cross-package symbol references via single-backtick stays; **same-module overload references** also use single-backtick prose form (DocC's `-some_overload` disambiguator is fragile).
- Mid-phase RFC authoring is welcome but not forced; the criterion-3 miss should be re-framed in future gate templates.
- **New:** the v0.2 migration sweep pattern is proven (RFC-0010 → executed cleanly in Phase 6). Future cross-cutting type additions can follow the same shape.
- **New:** the staging pattern "v0.1 = decompression / readonly half, v0.2 = compression / mutating half" is established for codecs (analogous to swift-uri's "v0.1 = ASCII-fast-path, v0.2 = full IDNA"). Phase 7 will exercise it heavily.

---

## Decision

Gate 6 is **PASSED on items 1, 2, 4** and **NOT YET PASSED on items 3 (failed) and 5 (open)**. Phase 7 plans should be drafted but not executed until item 5 closes via RFC-0012.

Phase 6 is the third consecutive phase to ship its 4D stretch and the largest tier wave to date. Item 3's miss is a process-criterion artifact, not a real failure — Phase 6 simply had no policy gaps to close mid-flight. The JSON tier shipped cleanly with 13 total format-tier packages now in the ecosystem; the v0.2 Time sweep cleanly closed RFC-0010's migration plan.

The next step is **RFC-0012 anchoring Phase 7 on HTTP body codecs / compression**, which both clears Gate 6 and structures the next wave.

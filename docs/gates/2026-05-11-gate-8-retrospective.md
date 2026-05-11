# Gate 8 Retrospective: Phase 8 → Phase 9

**Date:** 2026-05-11
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 8 (the HTTP request/response primitives wave anchored by [RFC-0013](../../rfcs/0013-phase-8-anchor-http-request-response-primitives.md)) against the Gate 1–7 criteria template and recommends whether Phase 9 should start. All four tranches (8A–8D) shipped — **fifth consecutive phase to ship its stretch** (4D, 5D, 6D, 7D, 8D). The chosen 8D was swift-link-header (RFC 8288); swift-brotli stays on the candidate list for future phases.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All anchored Tranche 8A/8B/8C/8D packages shipped at v0.1+, CI green, DocC live, listed | ✓ PASS |
| 2 | RFC-0013 commitments fulfilled (multipart / range / conditional / link-header) | ✓ PASS |
| 3 | ≥1 new RFC accepted during Phase 8 | ✗ NONE — third consecutive miss; reframing recommended |
| 4 | RFC-0001 conventions stress-test against Phase 8 packages | ✓ DONE BELOW |
| 5 | Phase 9 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0014 below) |

**Overall:** Items 1, 2, 4 pass cleanly. Item 3 fails for the **third consecutive gate** (Phase 6 / 7 / 8); the criterion as written is now operationally wrong. Item 5 is open. Phase 9 plans should be drafted but not executed until item 5 closes via RFC-0014.

**Item 3 has now missed three gates in a row.** Phase 6 (JSON tier), Phase 7 (compression), and Phase 8 (HTTP primitives) all executed without surfacing cross-cutting policy gaps that warranted a mid-flight RFC. The retrospective recommends formally **reframing item 3** in future templates to "≥1 substantive RFC accepted *or documented absence of policy gaps*" so phases of pure execution can pass cleanly.

The roadmap's stop conditions did not trigger: Phase 8 calendar time was ~1 day for all four tranches. No security incidents. No Apple announcements that subsumed planned packages. RFC-0001 held under the new HTTP tier additions.

---

## Item 1 — Phase 8 packages shipped ✓ PASS

| Tranche | Package | v0.1+ tag | Tests | DocC | Index | CI |
|---|---|---|---|---|---|---|
| 8A | swift-multipart | v0.1.0 | 21 | ✓ | ✓ | green |
| 8B | swift-range | v0.1.0 | 30 | ✓ | ✓ | green |
| 8C | swift-conditional | v0.1.0 | 39 | ✓ | ✓ | green |
| 8D | swift-link-header | v0.1.0 | 32 | ✓ | ✓ | green |

All four tranches shipped — the 8D stretch was selected per RFC-0013's offered choice between swift-link-header and swift-brotli. swift-link-header paired naturally with the rest of the wave (HTTP header parser, no new deps, ~430 LOC), kept scope bounded, and shipped same session as 8C. Brotli remains a separate-package candidate for a future phase.

**122 new tests across the four packages** — the highest single-phase test count to date (Phase 7 had 55).

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists **38 packages**. The new "http" tier on the site has 4 packages (multipart + range + conditional + link-header).

**Calendar time:** Phase 8 executed in ~1 day. swift-conditional (8C) was the densest single package this phase (~600 LOC across ETag + ETagList + HTTPDate + Evaluator); swift-multipart (8A) was ~500 LOC; swift-range (8B) was ~450 LOC; swift-link-header (8D) was ~430 LOC. None approached the swift-uri or swift-deflate density of prior phases.

---

## Item 2 — RFC-0013 commitments ✓ PASS

[RFC-0013](../../rfcs/0013-phase-8-anchor-http-request-response-primitives.md) committed to the HTTP request/response primitives wave with one stretch slot:

| RFC-0013 commitment | Outcome |
|---|---|
| Tranche 8A: swift-multipart (RFC 7578 form-data parser + serializer) | ✓ shipped |
| Tranche 8B: swift-range (RFC 7233 byte-range header + Content-Range + satisfiability) | ✓ shipped |
| Tranche 8C: swift-conditional (RFC 7232 ETag + If-* + § 6 evaluator) | ✓ shipped |
| Tranche 8D *(stretch)*: pick from swift-link-header / swift-brotli | ✓ shipped — picked swift-link-header |
| Foundation-free, non-Codable across the wave | ✓ preserved across all four packages |
| swift-conditional reuses swift-time's RFC 1123 codec for Last-Modified | ✓ |
| Compression-encoder v0.2 sweep deferred to Phase 9 | ✓ deferred (this retro recommends RFC-0014 to formalize) |

The RFC's "out of scope" list (streaming multipart, multipart/mixed, file-upload helpers, If-Range, HTTP/2 push) held in every package's v0.1 release.

Notable findings beyond the RFC's stated commitments:

- **swift-range module renamed to `HTTPRange`.** RFC-0013 didn't pre-specify the module name; during 8B implementation the default `Range` collided with stdlib `Range<Bound>` and broke disambiguation in consumer code. Renamed to `HTTPRange` — documented as a feedback memory for future packages whose obvious name shadows the stdlib.
- **swift-conditional `If-Range` formally deferred to v0.2.** RFC-0013 listed If-Range as out of scope for swift-range v0.1; during 8C planning the question recurred — If-Range lives at the swift-range × swift-conditional boundary. v0.2 of either package can pull it in once the integration shape is settled.
- **swift-link-header parameter name handling.** During 8D the parser initially stored the `*` ext-value suffix as part of the parameter name, causing the serializer to emit `title**=` for round-trips. Fixed by stripping `*` in the parser — the `*` is syntactic, indicating the `.extended` value variant; serializer adds it back. Public API stays clean: `firstParameter(named: "title")` works for both wire forms.
- **Pre-merge bugs caught and fixed** (consistent with prior phases):
  - swift-multipart's initial Disposition struct used `[(name: String, value: String)]` for parameters; Swift 6 doesn't synthesize Equatable for array-of-tuples in member position. Fixed by introducing a `Parameter` sub-struct. Saved as a transferable feedback memory.
  - swift-conditional's initial Calendar comparison used naive field-tuples and ignored time-zone offsets, which would 412 the client over a server-side parse mismatch. Fixed by routing through `Time.Calendar.toInstant()`.
  - swift-conditional's ETagList parser's `while !s.isEmpty` short-circuited on trailing-comma input. Fixed by switching to `while true` with explicit empty-after-separator reject — saved as a transferable parser pattern.

---

## Item 3 — ≥1 new RFC accepted during Phase 8 ✗ FAIL (third consecutive)

**Pre-Phase 8 (Gate 7 closeout):** RFC-0013 (Phase 8 anchor) accepted 2026-05-10 to clear Gate 7 item 5.

**During Phase 8 execution:** **zero RFCs accepted.**

Third consecutive miss on item 3 (Phase 6 / 7 / 8 all had no in-flight RFCs). The reason is identical across all three: the cross-cutting decisions during execution were either already covered by the anchoring RFC or small enough to live in CHANGELOG / DocC notes / memory entries rather than warrant an RFC.

**Candidates surfaced during Phase 8 execution that warrant RFCs (next gate's work):**

### RFC-0014 — Phase 9 anchor [the gating RFC for closing Gate 8]

Same shape as RFC-0005 / 0007 / 0008 / 0009 / 0011 / 0012 / 0013. Decision space mapped in item 5 below. **Strong default: compression-encoder v0.2 sweep**, which closes RFC-0012's deferred commitment and was explicitly named in Gate 7's recommendation.

### RFC-0015 *(deferred)* — Module naming when stdlib collisions are likely

**Trigger:** swift-range collided with stdlib `Range<Bound>` and was renamed to `HTTPRange` mid-implementation. Future packages whose obvious name shadows the stdlib (`Date`, `URL`, `Sequence`, `Result`, etc.) will hit the same friction.

**Proposal:** Codify a naming-convention check at scaffolding time. Options:
- (a) **Documentation only** — add a "names to avoid" list to RFC-0001 conventions and let the package author decide. Cheapest; relies on author memory.
- (b) **Linting** — extend `bare-swift new` to refuse module names matching a curated stdlib-shadow list. Pre-emptive; rejects only at scaffold time.
- (c) **No formal policy** — accept rename-mid-implementation as the cost of doing business. Status quo.

**Status:** swift-range was the first ecosystem package to hit this; the cost is small (one rename, one feedback memory). Defer to a future gate unless a second case appears.

### RFC-0016 *(deferred)* — Test-count expectations

**Trigger:** Phase 8 produced 122 tests across four packages (avg 30.5 per package), substantially higher than Phase 7's 55 (avg 13.75). The header-parser-heavy nature of Phase 8 inflated counts (lots of grammar branches). RFC-0001's test-parity policy speaks to *what to test* (mirror the upstream / spec coverage) but not *how many*. Three phases of escalating counts may warrant a minimum-coverage guideline.

**Status:** Counts are following code complexity, not goldplating. Defer.

### Recommendation

Author and accept **RFC-0014** as the Phase 9 anchor. Defer RFC-0015 / 0016 — single-case triggers don't justify the RFC machinery yet.

**Item-3 reframe (third consecutive miss):** revise future gate templates to read "≥1 substantive RFC accepted *or* documented absence of policy gaps". The criterion as written measures the wrong thing: phases of pure execution that don't surface new policy questions are healthy, not deficient.

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

Per-package review against RFC-0001 for the 4 Phase 8 packages:

| Convention | All four | Notes |
|---|---|---|
| Module name PascalCase, no `Swift` prefix | ✓ | `Multipart`, `HTTPRange`, `Conditional`, `LinkHeader` |
| Apache-2.0 + LLVM exception SPDX | ✓ | Every source file. |
| NOTICE crediting upstream | ✓ | All four credit IETF specs (RFC 7578 / 7233 / 7232 / 8288). |
| Swift 6.0+ tools version | ✓ | All four. |
| macOS platform floor | ✓ | All four at macOS 14. |
| Sendable-clean by default | ✓ | All public types `Sendable`. No `@unchecked` needed. |
| Strict concurrency in CI | ✓ | All four. |
| Single public error enum, typed throws | ✓ | `MultipartError` (5), `HTTPRangeError` (3), `ConditionalError` (3), `LinkHeaderError` (4). |
| Public APIs Foundation-free | ✓ | None expose Foundation types. |
| Repo skeleton from `bare-swift new` | ✓ | All four. |
| README tagline + ≤30-line example | ✓ | All four. |
| CHANGELOG with v0.1 entry | ✓ | All four. |
| CI green on macOS + Linux | ✓ | All four. |
| DocC bundle | ✓ | All four. |

### Deviations and findings

**1. New "http" tier introduced.** First new tier on the umbrella since Phase 7's compression-tier addition. The "http" tier now has 4 packages. The umbrella's tier-grouping logic absorbed it without changes — confirms the tier system scales.

**2. swift-conditional pulled in a swift-time dep.** First Phase 8 cross-tier dependency. swift-time was added in Phase 5; its RFC 1123 codec is the canonical Foundation-free HTTP-date parser. Reusing it here closes the loop on a planned consumer (RFC-0011 named HTTP headers as a Time consumer). Documented in swift-conditional's README.

**3. Pre-emptive Pages tag-policy now canonical.** The github-pages environment auto-creates when Pages is enabled (verified during 8D's swift-link-header repo setup). New canonical order for fresh repos: enable Pages → POST v* deployment-branch-policy → tag. Saves the rerun-after-failure cycle that's hit 13+ times across Phases 4–7. Saved as a memory update.

**4. Pre-merge bug pattern continues, with a new sub-pattern.** Phase 8's pre-merge bugs were both **API-shape bugs caught by the type system**, not algorithm bugs:
  - swift-multipart's array-of-tuples Equatable failure (Swift 6 synthesis limit).
  - swift-link-header's parameter-name-with-`*` collision with the serializer (shape mismatch).
  Phase 7's bugs were algorithm bugs (peekBits EOF, FNAME padding). Suggests header-parser packages tend to surface API-shape issues during implementation, whereas codec packages surface algorithm issues.

**5. The first true *parser-and-serializer-must-round-trip* phase.** All four Phase 8 packages have both halves: parse(header) → serialize(parsed) → equivalent. This forced design discipline (e.g. preserving parameter order, distinguishing token vs quoted, distinguishing extended-value-with-`*` vs regular). Saved as an emerging design pattern for future header-grammar packages.

### Verdict

RFC-0001 held cleanly under Phase 8 stress (introduction of a new tier, first cross-tier dep in the wave, four consecutive round-trip parsers/serializers).

---

## Item 5 — Phase 9 anchor decision ✗ NOT YET (recommendation)

Phase 9 anchor candidates surveyed:

| Wave | Packages | Risk | Adoption fit |
|---|---|---|---|
| **Compression encoders v0.2 sweep** | swift-deflate v0.2 + swift-gzip v0.2 + swift-zlib v0.2 + swift-content-encoding v0.2 | Medium-high (DEFLATE encoder ~1500 LOC) | **High** — closes RFC-0012's encoder commitment. Same audience as Phases 7+8. |
| Crypto-adjacent | swift-blake3, swift-siphash, swift-hkdf, swift-x25519 | Medium-low | Medium — sixth rejection if rejected again. swift-crypto remains entrenched. |
| Internationalization | swift-idna, swift-publicsuffix | Medium-high | Medium — closes swift-uri's IDNA deferral. Niche but real. |
| OTLP variants | swift-otlp-traces v0.2, swift-otlp-logs v0.2, swift-otlp-json | Medium | Medium — closes Phase 2/3/4 cross-signal deferral. |
| Auth primitives | swift-jwt-verify, swift-basic-auth, swift-bearer | Medium | Medium-high — natural Phase-8 follow-on. |
| HTTP body v0.2 | swift-multipart streaming, swift-range If-Range integration | Low-medium | Medium — closes Phase 8 deferrals. |

### Recommendation: **Anchor on the compression-encoder v0.2 sweep**

Reasons:

1. **Closes a committed deferral.** RFC-0012 explicitly committed all four compression-tier packages to a v0.2 release adding the encoder. Phase 7 retro deferred *when* to Gate 8. Gate 8 is now. Picking anything else extends the deferral indefinitely.
2. **DEFLATE encoder is anchor-worthy on its own.** ~1500 LOC of LZ77 match-finding + canonical-Huffman tree construction + dynamic block selection. Larger than any Phase 8 single package; comparable to swift-deflate's INFLATE decoder (~600 LOC) plus what it took to make INFLATE production-ready.
3. **Audience continuity is total.** Phases 4 / 7 / 8 all serve HTTP server/client implementers. v0.2 encoders complete the compression story they already use.
4. **Builds on Phase 7 foundations.** swift-deflate's BitReader / canonical-Huffman tables are designed for encoder reuse; gzip / zlib / content-encoding are thin wrappers around the encoder once it lands.
5. **Risk profile is well-understood.** DEFLATE encoding is famously a tuning problem (`zopfli` vs `gzip -9` vs `gzip -6` produces output of identical correctness but varying size). v0.2 commits to *correctness*; size/speed tuning lands as v0.2.x patch releases.
6. **Sets up Phase 10+ options.** Once compression encoders ship:
   - **swift-brotli** becomes the natural Phase 10 anchor (the deferred candidate from Phases 7 & 8).
   - **Auth primitives** become approachable as a focused mini-phase.
   - **OTLP v0.2 cross-signal** closes the observability deferral.

### Phase 9 tranche sketch (subject to formal RFC)

| Tranche | Packages | Estimated calendar |
|---|---|---|
| 9A | swift-deflate v0.2 — RFC 1951 encoder (LZ77 + canonical Huffman + dynamic blocks) — the headline | ~2–3 days at observed pace |
| 9B | swift-gzip v0.2 — RFC 1952 encoder (header serialization + CRC-32 + ISIZE) | ~0.5 day |
| 9C | swift-zlib v0.2 — RFC 1950 encoder (header serialization + ADLER32) | ~0.5 day |
| 9D *(stretch)* | swift-content-encoding v0.2 (encoder pipeline) OR swift-brotli (decoder) OR swift-otlp-traces v0.2 | ~1–2 days |

Total Phase 9 budget: 1 week calendar at observed pace; substantially smaller than Phase 8 in package-count terms (3 baseline + 1 stretch instead of 4 distinct) but **larger in LOC terms** because the DEFLATE encoder dominates.

### Why other waves were rejected

- **Crypto-adjacent** — sixth rejection candidate. swift-crypto remains entrenched. Differentiation hasn't strengthened across six gates.
- **Internationalization** — niche compared to closing the encoder commitment. Better as a Phase 10+ side deliverable.
- **OTLP variants** — fits a documented deferral but the audience is narrower than encoder completion. Better timing once Phase 9 ships.
- **Auth primitives** — natural Phase 8 follow-on but lower priority than honoring the RFC-0012 commitment. Strong Phase 10 candidate.
- **HTTP body v0.2** — small enough to land as patch releases when consumers ask; not anchor-worthy.

**Action:** author the anchor decision as **RFC-0014** (Phase 9 anchor: compression-encoder v0.2 sweep) and accept it before Phase 9 plans start.

---

## Open work to clear Gate 8

1. **Write RFC-0014** (Phase 9 anchor: compression-encoder v0.2 sweep). 1–2 hour task. References this retrospective and RFC-0012.
2. **Begin Phase 9 planning** after RFC-0014 accepts.

**Recommended cadence:** RFC-0014 immediately after this retro is read → Phase 9 planning starts → first Tranche 9A package (swift-deflate v0.2) within a session.

---

## What this retrospective changes for Phase 9

- The plan-then-execute-with-checkpoints workflow stays.
- Inline-vector test pattern stays; encoder validation will use **round-trip-then-zlib-decompress** against reference inputs (encode with our DEFLATE → decompress with `python -m zlib.decompress` → compare).
- `docc-target` opt-in upfront stays canonical.
- **GitHub Pages two-step setup is now one step.** Tag deployment-policy can be applied *before* the first tag push (verified in swift-link-header). New canonical order for fresh repos. Saves the rerun-after-failure cycle that's hit 13+ times across Phases 4–7.
- Cross-package symbol references via single-backtick stays.
- **New:** when an existing package's API is being extended in a minor (v0.1 → v0.2), prefer **additive** extensions over breaking changes. The decoder API in v0.1 must remain intact in v0.2 — the encoder adds a parallel API surface, doesn't restructure the existing one.
- **New:** parser-and-serializer round-trip discipline is now an ecosystem-canonical pattern for header-grammar packages (verified across all four Phase 8 packages). Future header packages should design parse + serialize as a unit, not parse first and serialize later.
- **Item-3 criterion needs formal reframing.** Three consecutive misses (Phase 6 / 7 / 8) suggest the criterion as written is operationally wrong; future gate templates should treat "documented absence of policy gaps" as equivalent to a substantive RFC. This retrospective formally recommends the reframe.

---

## Decision

Gate 8 is **PASSED on items 1, 2, 4** and **NOT YET PASSED on items 3 (failed third consecutive gate; reframe recommended) and 5 (open)**. Phase 9 plans should be drafted but not executed until item 5 closes via RFC-0014.

Phase 8 is the **fifth consecutive phase to ship its 4D stretch** and the second to introduce a new tier (http) in two phases. The Phase 8 packages deliver every primitive a Foundation-free HTTP server / client needs for request-body parsing (multipart), partial-content responses (range), conditional caching (conditional), and hypermedia links (link-header) — when combined with Phase 4's URI / MIME / cookie packages and Phase 7's compression, the bare-swift ecosystem now covers the HTTP request/response surface with 16 packages across three tiers.

The next step is **RFC-0014 anchoring Phase 9 on the compression-encoder v0.2 sweep**, which both clears Gate 8 and honors the explicit deferral from RFC-0012. swift-brotli, internationalization, OTLP variants, and auth primitives remain on the queue for Phase 10+.

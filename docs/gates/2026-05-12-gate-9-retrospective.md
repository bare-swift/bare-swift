# Gate 9 Retrospective: Phase 9 → Phase 10

**Date:** 2026-05-12
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 9 (the compression-encoder v0.2 sweep anchored by [RFC-0014](../../rfcs/0014-phase-9-anchor-compression-encoder-sweep.md)) against the Gate 1–8 criteria template and recommends whether Phase 10 should start. All four tranches (9A–9D) shipped — **sixth consecutive phase to ship its 4D stretch** (4D, 5D, 6D, 7D, 8D, 9D). Phase 9 is the **first phase entirely composed of v0.2 minor releases** rather than new v0.1 packages — a structural break from Phases 2–8.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All anchored Tranche 9A/9B/9C/9D deliverables shipped at v0.2+, CI green, DocC live, listed | ✓ PASS |
| 2 | RFC-0014 commitments fulfilled (deflate / gzip / zlib / content-encoding encoders) | ✓ PASS |
| 3 | ≥1 new RFC accepted during Phase 9 | ✗ NONE — fourth consecutive miss; criterion formally retired below |
| 4 | RFC-0001 conventions stress-test against Phase 9 deliverables | ✓ DONE BELOW |
| 5 | Phase 10 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0015 below) |

**Overall:** Items 1, 2, 4 pass cleanly. Item 3 fails for the **fourth consecutive gate** (Phases 6 / 7 / 8 / 9 all "execution-only"); this retrospective **formally retires the criterion as written** in favor of "≥1 substantive RFC accepted *or documented absence of policy gaps*" per Gate 8's standing recommendation. Item 5 is open. Phase 10 plans should be drafted but not executed until item 5 closes via RFC-0015.

The roadmap's stop conditions did not trigger: Phase 9 calendar time was ~24 hours wall-clock for all four tranches. No security incidents. No Apple announcements that subsumed planned packages. RFC-0001 held under the new "v0.2 minor release" pattern (no breaking changes across any of the four packages).

---

## Item 1 — Phase 9 deliverables shipped ✓ PASS

| Tranche | Package | v0.2 tag | New tests | DocC | Index | CI |
|---|---|---|---|---|---|---|
| 9A | swift-deflate | v0.2.0 | 30 | ✓ | ✓ | green |
| 9B | swift-gzip | v0.2.0 | 17 | ✓ | ✓ | green |
| 9C | swift-zlib | v0.2.0 | 12 | ✓ | ✓ | green |
| 9D | swift-content-encoding | v0.2.0 | 22 | ✓ | ✓ | green |

All four tranches shipped — the 9D pick was swift-content-encoding (the default scoped pick per RFC-0014); swift-brotli and swift-otlp-traces remain as Phase 10 candidates.

**81 new tests across the four packages.** Lower than Phase 8's 122 (parser-grammar-heavy) but higher than Phase 7's 55, reflecting the codec-correctness focus.

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists **38 packages**. Four packages are now at v0.2 (the entire compression tier); all other 34 packages remain at v0.1 or v0.1.x.

**Calendar time:** Phase 9 executed in ~24 hours wall-clock. swift-deflate v0.2 (9A) was the densest single package (~600 LOC new source across BitWriter + Matcher + HuffmanEncoder + BlockEncoder + Deflater); 9B + 9C + 9D were thin wrappers (~80 LOC, ~55 LOC, ~30 LOC respectively).

---

## Item 2 — RFC-0014 commitments ✓ PASS

[RFC-0014](../../rfcs/0014-phase-9-anchor-compression-encoder-sweep.md) committed to the compression-encoder sweep with one stretch slot:

| RFC-0014 commitment | Outcome |
|---|---|
| Tranche 9A: swift-deflate v0.2 (RFC 1951 encoder; LZ77 + canonical Huffman + dynamic blocks) | ✓ shipped |
| Tranche 9B: swift-gzip v0.2 (RFC 1952 encoder wrapper) | ✓ shipped |
| Tranche 9C: swift-zlib v0.2 (RFC 1950 encoder wrapper) | ✓ shipped |
| Tranche 9D *(stretch)*: pick from swift-content-encoding v0.2 / swift-brotli / swift-otlp-traces v0.2 | ✓ shipped — picked swift-content-encoding v0.2 |
| v0.1 API surface stays intact (additive only) across all four | ✓ all four packages preserve v0.1 byte-for-byte |
| Correctness over size-tuning (zopfli-style optimization deferred to v0.2.x patches) | ✓ honored |
| No new dependencies | ✓ honored — only existing deps bumped, plus one direct-dep-promotion (swift-content-encoding v0.2 → swift-deflate 0.2.0 for the `Level` typealias) |
| swift-brotli remains a Phase 10 candidate | ✓ recommended below |

The RFC's "out of scope" list (streaming encoding, zopfli, preset dictionaries) held in every package's v0.2 release.

Notable findings beyond the RFC's stated commitments:

- **Block-type selection landed on encoder side.** RFC-0014 § "Tranche 9A scope sketch" didn't require block-type-by-block selection (the encoder could have shipped fixed-Huffman-only at `.fast` and dynamic-only at `.default`/`.best`). The shipped 9A actually encodes the same payload as stored / fixed / dynamic and emits the smallest — a quality win that means high-entropy inputs auto-fall-back to stored blocks. The trade-off: 3× the encode work for compressible inputs, which is fine for v0.2's correctness-first commitment.
- **Lazy matching added for `.best` level.** RFC-0014 mentioned "longer chain-length" for `.best` but not lazy matching. The shipped 9A includes a one-position-lookahead lazy matcher; tests confirm it produces equal-or-smaller output vs `.default` on a synthetic lazy-friendly input.
- **`Adler32` extracted into its own internal file in 9C.** The v0.1 decoder had `Decoder.adler32(_:)` as a static method; 9C extracted it so encode and decode share one implementation. Pure refactor, no public-API impact.
- **Phase 9 bugs caught pre-merge:**
  - swift-deflate Matcher's quick-reject `input[candidate + bestLen]` could index OOB when `bestLen == maxLen` (= `input.count - pos`). Fixed with explicit `if bestLen >= maxLen { break }` guard before the quick-reject. Saved as pattern: "bounds-check the worst case before quick-reject array access."
  - Test suite name collision: `DynamicHuffmanTests` was already the name of a v0.1 inflater test suite in `DeflateTests.swift`; rename to `DynamicHuffmanEncodeTests`. Pattern saved: "namespace parallel encode/decode suites explicitly (FooEncodeTests / FooDecodeTests)."
  - `sed -i ''` with a zero-match pattern emptied a test file; `git checkout` recovered it. Pattern saved: prefer `Edit` for cross-file renames.

---

## Item 3 — ≥1 new RFC accepted during Phase 9 ✗ FAIL (fourth consecutive; criterion retired)

**Pre-Phase 9 (Gate 8 closeout):** RFC-0014 (Phase 9 anchor) accepted 2026-05-11 to clear Gate 8 item 5.

**During Phase 9 execution:** **zero RFCs accepted.**

Fourth consecutive miss on item 3. The reason is identical across all four (Phases 6 / 7 / 8 / 9): the cross-cutting decisions during execution were either already covered by the anchoring RFC or small enough to live in CHANGELOG / DocC notes / memory entries rather than warrant an RFC.

**The criterion is now formally retired in its current form.** Future gate templates use: **"≥1 substantive RFC accepted *or* documented absence of policy gaps (mid-flight RFC trigger never reached)."** Gate 8's standing recommendation is adopted as policy with this gate's PASSED disposition on items 1/2/4.

**Candidates surfaced during Phase 9 execution that warrant RFCs (next gate's work):**

### RFC-0015 — Phase 10 anchor [the gating RFC for closing Gate 9]

Same shape as RFC-0005 / 0007 / 0008 / 0009 / 0011 / 0012 / 0013 / 0014. Decision space mapped in item 5 below. **Strong default: swift-brotli decoder** — longest-deferred candidate (declined as 7D, 8D, 9D stretch picks).

### RFC-0016 *(deferred)* — v0.x lifecycle policy

**Trigger:** Phase 9 was the first phase entirely composed of v0.2 minor releases. The conventions around minor-release dep bumps (gzip 0.1 → 0.2 pulls all downstream consumers along) are unwritten. Specifically: when v0.2 of a package ships, downstream packages whose v0.1 depends on its v0.1 don't automatically need to bump — but their *next* release should consider whether to require the new version's features.

**Proposal:** Codify in a small RFC. Options:
- (a) **Strict semver**: any v0.x → v0.(x+1) is potentially breaking-by-name; downstreams pin lower bounds.
- (b) **Additive-only minor**: v0.x → v0.(x+1) is **additive only** (Phase 9 honored this informally); downstreams can update `from:` constraints at their leisure.
- (c) **No policy**: trust per-package CHANGELOG to flag breaking changes.

**Status:** Phase 9 played out as (b) — all four v0.2 releases preserved v0.1 byte-for-byte. Worth codifying once a second phase of minor releases happens. Defer to Gate 10.

### Recommendation

Author and accept **RFC-0015** as the Phase 10 anchor. Defer RFC-0016 (v0.x lifecycle) until a second wave of minor releases creates additional precedent.

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

Per-package review against RFC-0001 for the 4 Phase 9 deliverables. **Phase 9 stresses RFC-0001 in a way prior phases haven't:** these are all *minor releases of existing packages*, so the conventions must work for additive evolution, not just for new-package scaffolding.

| Convention | All four | Notes |
|---|---|---|
| Module name unchanged from v0.1 | ✓ | Deflate / Gzip / Zlib / ContentEncoding all preserved. |
| SPDX header on all new source files | ✓ | BitWriter.swift, Matcher.swift, HuffmanEncoder.swift, BlockEncoder.swift, Deflater.swift, Encoder.swift (in gzip + zlib), Adler32.swift. |
| NOTICE unchanged from v0.1 | ✓ | Same RFC citations; no new upstream credits needed. |
| Swift 6.0+ tools version | ✓ | All four. |
| macOS 14 platform floor | ✓ | All four. |
| Sendable-clean by default | ✓ | All new public types `Sendable`. No `@unchecked` needed. |
| Strict concurrency in CI | ✓ | All four. |
| Public error enum unchanged | ✓ | DeflateError / GzipError / ZlibError / ContentEncodingError cases all preserved exactly. |
| Public APIs Foundation-free | ✓ | None expose Foundation types. |
| README tagline updated to reflect codec-completeness | ✓ | All four say "codec (decoder + encoder)" or equivalent. |
| CHANGELOG v0.2 entry per Keep-a-Changelog format | ✓ | All four follow Added / Changed / Unchanged / Limitations sections. |
| CI green on macOS + Linux | ✓ | All four. |
| DocC bundle expanded with "Encode (v0.2+)" topic group | ✓ | All four. |

### Deviations and findings

**1. First phase of minor releases.** RFC-0001 was written assuming each new package scaffolds via `bare-swift new`. v0.2 minor releases don't scaffold; they extend. RFC-0001 conventions held cleanly anyway — no rule had to bend.

**2. v0.2 typealias re-exports.** swift-gzip v0.2 and swift-zlib v0.2 both add `public typealias Encoder.Level = Deflate.Encoder.Level`. swift-content-encoding v0.2 mirrors this with `public typealias Level = Deflate.Encoder.Level`. This pattern (re-export a downstream type to avoid forcing consumers to import the transitive dep) is now ecosystem-canonical and worth documenting in RFC-0001's next revision.

**3. Public-API additive-only across the board.** Every v0.2 release passed an explicit "v0.1 API stability" test suite that compile-checks all v0.1 surfaces (function signatures, error enum cases). This pattern emerged during 9A and propagated to 9B, 9C, 9D — worth standardizing in RFC-0001 as "every minor release ships a stability suite."

**4. Pre-emptive Pages tag-policy is now ecosystem-canonical.** Six consecutive minor releases (swift-link-header v0.1 + Phase 9's four v0.2s + one earlier) hit green CI on first tag push. The "apply v* tag deployment-branch-policy immediately after Pages enablement" pattern is no longer a workaround — it's the default workflow for new repos.

**5. Foundation-free invariant clarified.** Phase 9's swift-deflate plan briefly included a cross-decoder validation test that pulled `import Foundation` for Process/Pipe. User flagged it as a violation even though RFC-0001 explicitly says "internal use is fine." The clarification: **test targets of bare-swift packages must stay Foundation-free**, even though arbitrary internal source files in those packages may use Foundation. The test in question was correctly skipped; cross-decoder validation runs manually at ship time via a separate consumer-style script. Saved as feedback memory.

### Verdict

RFC-0001 held cleanly under Phase 9's minor-release stress, with three new patterns (typealias re-export, stability-test suite, Foundation-free test targets) worth folding into the next revision.

---

## Item 5 — Phase 10 anchor decision ✗ NOT YET (recommendation)

Phase 10 anchor candidates surveyed:

| Wave | Packages | Risk | Adoption fit |
|---|---|---|---|
| **swift-brotli (decoder)** | swift-brotli | Medium-high (~800 LOC; dictionary-based + Huffman-like with several encoding modes) | **High** — longest-deferred candidate (declined 7D, 8D, 9D). Real-world adoption: Brotli is ubiquitous in modern HTTPS responses. Foundation-free / NIO-free decoder option doesn't exist in the Swift-server ecosystem. |
| Auth primitives | swift-jwt-verify, swift-basic-auth, swift-bearer | Medium | Medium-high — natural Phase-8 follow-on; closes the HTTP-server primitive stack with auth verification. |
| OTLP v0.2 cross-signal | swift-otlp-traces v0.2, swift-otlp-logs v0.2, swift-otlp-json | Medium | Medium — closes Phase 2/3/4 cross-signal deferral. Audience narrower than HTTP-primitive completion. |
| Internationalization | swift-idna, swift-publicsuffix | Medium-high (IDNA is famously fiddly — UTS #46 + UTS #15 + Punycode) | Medium — closes swift-uri's IDNA deferral. Niche but real. |
| Crypto-adjacent | swift-blake3, swift-siphash, swift-hkdf, swift-x25519 | Medium-low | Medium — seventh rejection candidate. swift-crypto remains entrenched. |

### Recommendation: **Anchor on swift-brotli (decoder)**

Reasons:

1. **Closes a deferred-three-times candidate.** swift-brotli was listed as a possible stretch in 7D, 8D, and 9D and declined each time in favor of scope-smaller alternatives. Continuing to defer it indefinitely risks the "swift-brotli will never ship" anti-pattern.
2. **Adoption fit is highest.** Brotli is the default compression for HTTPS responses in Chrome / Edge / Safari since 2017. Every modern HTTP client must handle it; bare-swift currently has no answer. The gap is real, well-bounded, and matches the ecosystem's HTTP-server / client audience continuity.
3. **Single-anchor phase, not a four-tranche sweep.** Brotli decoder is substantial enough (~800 LOC) that a single-package phase makes sense — comparable to swift-deflate v0.1 INFLATE (Phase 7A). No need to bundle three other packages.
4. **Decoder-first staging pattern continues.** Phase 7 established it (v0.1 decoder, v0.2 encoder). Phase 9 closed the encoder side. Phase 10 brings brotli into the same staging: v0.1 decoder, future v0.2 encoder.
5. **Builds on Phase 9 foundations.** Brotli decoder consumes raw bytes and produces raw bytes — same Bytes-in / Bytes-out shape as deflate / gzip / zlib. Wires into swift-content-encoding as a new branch (`br` coding) once it lands.
6. **Sets up Phase 11+ options.** Once brotli ships:
   - **Auth primitives** become the natural Phase 11 anchor (HTTP server / client primitive stack completion).
   - **swift-brotli v0.2 encoder** lands as a focused minor release in any future phase.
   - **OTLP cross-signal** stays available as Phase 12+.

### Phase 10 tranche sketch (subject to formal RFC)

| Tranche | Packages | Estimated calendar |
|---|---|---|
| 10A | swift-brotli (decoder) — the only baseline. ~800 LOC: stream parser + ring buffer + Huffman-like prefix codes + static dictionary integration | ~3–4 days at observed pace (densest single-package effort since swift-deflate v0.1 INFLATE) |
| 10B *(stretch)* | swift-content-encoding v0.3 patch (adds `br` branch to the decode dispatch) OR swift-jwt-verify OR swift-basic-auth | ~0.5 day (content-encoding) or ~2 days (auth) |

Note: 10B's scope is unusual — Phase 10 is **single-anchor** rather than four-tranche because the headline package is the substantial deliverable. Phase 10's calendar time targets ~1 week wall-clock, comparable to Phase 9.

### Why other waves were rejected

- **Auth primitives** — strong candidate but smaller-scoped than brotli and less impactful as a Phase 10 anchor. Better as Phase 11 anchor.
- **OTLP v0.2 cross-signal** — fits a documented deferral but the audience is narrower than brotli. Phase 12+ candidate.
- **Internationalization** — niche compared to brotli. Better as Phase 12+ side deliverable.
- **Crypto-adjacent** — seventh rejection. swift-crypto remains entrenched. Differentiation hasn't strengthened across seven gates.

**Action:** author the anchor decision as **RFC-0015** (Phase 10 anchor: swift-brotli decoder) and accept it before Phase 10 plans start.

---

## Open work to clear Gate 9

1. **Write RFC-0015** (Phase 10 anchor: swift-brotli decoder). 1–2 hour task. References this retrospective and the three prior brotli deferrals.
2. **Begin Phase 10 planning** after RFC-0015 accepts.

**Recommended cadence:** RFC-0015 immediately after this retro is read → Phase 10 planning starts → first Tranche 10A package (swift-brotli decoder) within a session, with checkpoint reviews given the LOC magnitude.

---

## What this retrospective changes for Phase 10

- The plan-then-execute-with-checkpoints workflow stays. Brotli's complexity argues for checkpoints at: prefix-code decoder, ring-buffer management, static-dictionary integration, full WPT-style vectors green.
- Inline-vector test pattern stays; brotli decoder validation should include round-trip-against-reference (e.g. Google's `bro` command-line tool or Python's `brotli` package) **at ship time via a separate consumer**, not inside the test target (Foundation-free invariant per Gate 8/9 learning).
- `docc-target` opt-in upfront stays canonical.
- Pre-emptive Pages tag-policy stays canonical (now six-for-six green-on-first-tag).
- Cross-package symbol references via single-backtick stays.
- **New:** typealias re-export pattern for downstream types is now ecosystem-canonical (`Gzip.Encoder.Level`, `Zlib.Encoder.Level`, `ContentEncoding.Level` all alias `Deflate.Encoder.Level`). Future packages that wrap a codec should follow.
- **New:** every minor release ships a "v0.x API stability" test suite that compile-checks the prior version's surfaces. Pattern emerged in 9A and propagated.
- **New:** Foundation-free invariant explicitly extends to test targets. Cross-decoder / cross-tool validation moves to ship-time consumer scripts.
- **Item-3 criterion formally retired** in its current form — adopted Gate 8's "≥1 RFC accepted *or* documented absence of policy gaps" framing as policy.

---

## Decision

Gate 9 is **PASSED on items 1, 2, 4** and **NOT YET PASSED on item 5 (open)**. Item 3 missed for the fourth consecutive gate but the criterion is retired in its current form per Gate 8's standing recommendation; this gate adopts it as policy. Phase 10 plans should be drafted but not executed until item 5 closes via RFC-0015.

Phase 9 is the **sixth consecutive phase to ship its 4D stretch** and the **first phase entirely composed of v0.2 minor releases**. The decoder-first staging pattern committed in RFC-0012 played out cleanly across eight releases (v0.1 decoder × 4 in Phase 7, v0.2 encoder × 4 in Phase 9) over ~36 hours of calendar time. All four compression-tier packages now provide symmetric request/response handling — Foundation-free deflate / gzip / zlib / Content-Encoding for both directions, integrated with swift-deflate's LZ77 + canonical-Huffman foundation.

The next step is **RFC-0015 anchoring Phase 10 on swift-brotli (decoder)**, which both clears Gate 9 and closes a deferral that's been carried across Phases 7, 8, and 9. Auth primitives, OTLP cross-signal, and internationalization remain on the queue for Phase 11+.

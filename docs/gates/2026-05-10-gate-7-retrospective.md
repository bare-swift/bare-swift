# Gate 7 Retrospective: Phase 7 → Phase 8

**Date:** 2026-05-10
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 7 (the compression / HTTP body codecs wave anchored by [RFC-0012](../../rfcs/0012-phase-7-anchor-http-body-codecs.md)) against the Gate 1–6 criteria template and recommends whether Phase 8 should start. All four tranches (7A–7D) shipped — fourth consecutive phase to ship its stretch (4D, 5D, 6D, 7D). The "decoders v0.1, encoders v0.2" staging pattern committed in RFC-0012 held cleanly across all four packages.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All anchored Tranche 7A/7B/7C/7D packages shipped at v0.1+, CI green, DocC live, listed | ✓ PASS |
| 2 | RFC-0012 commitments fulfilled (deflate / gzip / zlib / content-encoding) | ✓ PASS |
| 3 | ≥1 new RFC accepted during Phase 7 | ✗ NONE — see below |
| 4 | RFC-0001 conventions stress-test against Phase 7 packages | ✓ DONE BELOW |
| 5 | Phase 8 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0013 below) |

**Overall:** Items 1, 2, 4 pass cleanly. Item 3 fails for the second consecutive gate (also failed in Gate 6). Item 5 is open. Phase 8 plans should be drafted but not executed until item 5 closes via RFC-0013.

**Item 3 has now missed two gates in a row.** Both Phase 6 (JSON tier) and Phase 7 (compression) executed without surfacing any cross-cutting policy gaps that warranted a mid-flight RFC. The criterion as written wants the RFC machinery to exercise during execution, but operationally that's only valuable when the trigger is genuine. Two consecutive misses suggests reframing: **future gate templates may treat item 3 as "≥1 substantive RFC accepted *or documented absence of policy gaps*"** so phases that are pure execution can pass cleanly.

The roadmap's stop conditions did not trigger: Phase 7 calendar time was ~1 day for all four tranches. No security incidents. No Apple announcements that subsumed planned packages. RFC-0001 held under the new compression tier.

---

## Item 1 — Phase 7 packages shipped ✓ PASS

| Tranche | Package | v0.1+ tag | Tests | DocC | Index | CI |
|---|---|---|---|---|---|---|
| 7A | swift-deflate | v0.1.0 | 11 | ✓ | ✓ | green |
| 7B | swift-gzip | v0.1.0 | 13 | ✓ | ✓ | green |
| 7C | swift-zlib | v0.1.0 | 14 | ✓ | ✓ | green |
| 7D | swift-content-encoding | v0.1.0 | 17 | ✓ | ✓ | green |

All four tranches shipped — the 7D stretch was selected per RFC-0012's offered choice between swift-content-encoding and swift-brotli. swift-content-encoding paired naturally with the rest of the wave (composes 7B + 7C), kept scope bounded, and shipped same session as 7C. Brotli remains a separate-package candidate for a future phase.

**55 new tests across the four packages.**

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists 34 packages. The new "compression" tier on the site has 4 packages.

**Calendar time:** Phase 7 executed in ~1 day. swift-deflate (7A) was the densest single package this phase (~600 LOC INFLATE implementation: BitReader + canonical-Huffman decoder with table replication + block-type dispatch). The gzip / zlib / content-encoding tranches were all small wrappers (~150 LOC each).

---

## Item 2 — RFC-0012 commitments ✓ PASS

[RFC-0012](../../rfcs/0012-phase-7-anchor-http-body-codecs.md) committed to the compression wave with one stretch slot:

| RFC-0012 commitment | Outcome |
|---|---|
| Tranche 7A: swift-deflate (RFC 1951 INFLATE) | ✓ shipped |
| Tranche 7B: swift-gzip (RFC 1952 wrapper) | ✓ shipped |
| Tranche 7C: swift-zlib (RFC 1950 wrapper) | ✓ shipped |
| Tranche 7D *(stretch)*: pick from swift-content-encoding / swift-brotli | ✓ shipped — picked swift-content-encoding |
| Decoders v0.1, encoders deferred to v0.2 (staging pattern) | ✓ preserved across all four packages |
| Foundation-free, non-Codable across the wave | ✓ preserved across all four packages |
| swift-deflate uses swift-bytes for input/output | ✓ |
| swift-gzip uses swift-crc for CRC-32/ISO-HDLC | ✓ |

The RFC's "out of scope" list (DEFLATE encoder, multi-member streams, FHCRC validation, preset dictionary, brotli, zstd) held in every package's v0.1 release.

Notable findings beyond the RFC's stated commitments:

- **swift-zlib skipped swift-crc.** RFC-0012 didn't pre-commit a CRC dependency for zlib; the implementation noted that ADLER32 is a Fletcher-style checksum (not a CRC) and shipped an inline ~10-line implementation per RFC 1950 § 9. Reduced the dep graph; documented in the package's NOTICE.
- **swift-content-encoding skipped swift-mime.** The `Content-Encoding` header value is a simple comma-separated token list — much simpler than swift-mime's `type/subtype; param=value` grammar. An inline header parser kept the multiplexer focused and avoided a transitive dep.
- **Pre-merge bugs caught and fixed** (consistent with prior phases):
  - swift-deflate's initial `peekBits` threw on EOF — broke decoding of streams whose final symbol's bits were the actual final bits. Fixed by zero-padding the peek and pushing the EOF check to `consume`. Documented as a Huffman-table-replication property: shorter codes are replicated across all high-bit suffixes, so zero-padding the peek doesn't change the lookup result.
  - swift-gzip's initial `unterminatedFNAME` test padded with zero bytes — but those ARE NULs, so the parser terminated the FNAME prematurely. Fixed by padding with `0x41` so the parser scans to true EOF.

---

## Item 3 — ≥1 new RFC accepted during Phase 7 ✗ FAIL

**Pre-Phase 7 (Gate 6 closeout):** RFC-0012 (Phase 7 anchor) accepted 2026-05-10 to clear Gate 6 item 5.

**During Phase 7 execution:** **zero RFCs accepted.**

Second consecutive miss on item 3 (Phase 6 also had no in-flight RFCs). The reason is identical: the cross-cutting decisions during Phase 7 (decoder-first staging, header-format choices, swift-crc vs inline ADLER32) were either already covered by RFC-0012 or were small enough to live in CHANGELOG / DocC notes rather than warrant an RFC.

**Candidates surfaced during Phase 7 execution that warrant RFCs (next gate's work):**

### RFC-0013 — Phase 8 anchor [the gating RFC for closing Gate 7]

Same shape as RFC-0005 / 0007 / 0008 / 0009 / 0011 / 0012. Decision space mapped in item 5 below.

### RFC-0014 *(deferred)* — Compression-encoder v0.2 sweep timing

**Trigger:** RFC-0012 committed all four compression-tier packages to a v0.2 minor release that adds the encoder. RFC-0012 didn't commit *when* — only that the encoder lands "once the decoder is stable."

**Proposal:** Codify when. Options:
- (a) **Phase 8 parallel work-stream** — encoder sweep alongside whatever Phase 8 anchors on (mirrors RFC-0011's Time v0.2 sweep pattern).
- (b) **Phase 9 anchor** — encoders are substantial enough to anchor on (DEFLATE encoder ~1500 LOC; gzip/zlib/content-encoding are thin wrappers).
- (c) **On demand** — ship v0.2 of each whenever a real consumer asks for the encoder side.

**Status:** the multi-package nature of the sweep (4 packages, 1 substantial encoder) means it's anchor-worthy. RFC-0014 would commit option (b). Defer until Phase 8 anchor decision is final.

### Recommendation

Author and accept **RFC-0013** as the Phase 8 anchor. Skip RFC-0014 in this gate — Phase 8 anchor will implicitly resolve the encoder timing.

The item-3 miss should not be "fixed" by forcing an RFC. **Future gate templates should reframe item 3** as "≥1 substantive RFC accepted *or* documented absence of policy gaps" so phases of pure execution can pass cleanly. This is the second consecutive miss; the criterion needs revision.

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

Per-package review against RFC-0001 for the 4 Phase 7 packages:

| Convention | All four | Notes |
|---|---|---|
| Module name PascalCase, no `Swift` prefix | ✓ | `Deflate`, `Gzip`, `Zlib`, `ContentEncoding` |
| Apache-2.0 + LLVM exception SPDX | ✓ | Every source file. |
| NOTICE crediting upstream | ✓ | All four credit IETF specs (RFC 1951 / 1952 / 1950 / 9110). |
| Swift 6.0+ tools version | ✓ | All four. |
| macOS platform floor | ✓ | All four at macOS 14. |
| Sendable-clean by default | ✓ | All public types `Sendable`. No `@unchecked` needed. |
| Strict concurrency in CI | ✓ | All four. |
| Single public error enum, typed throws | ✓ | `DeflateError` (8), `GzipError` (9), `ZlibError` (8), `ContentEncodingError` (2). |
| Public APIs Foundation-free | ✓ | None expose Foundation types. |
| Repo skeleton from `bare-swift new` | ✓ | All four. |
| Tests/Vectors/EXCEPTIONS.md | ✓ | All four. |
| README tagline + ≤30-line example | ✓ | All four. |
| CHANGELOG with v0.1 entry | ✓ | All four. |
| CI green on macOS + Linux | ✓ | All four. |
| DocC bundle | ✓ | All four. |

### Deviations and findings

**1. Compression tier introduced.** First new tier on the umbrella since Phase 5's foundation-tier addition (swift-time). The "compression" tier now has 4 packages. The umbrella's tier-grouping logic absorbed it without changes — confirms the tier system scales to new categories.

**2. Decoder-first staging held cleanly.** RFC-0012's "v0.1 = decompression / readonly half, v0.2 = compression / mutating half" pattern shipped consistently across all four packages. Each CHANGELOG documents the encoder as v0.2. swift-deflate's BitReader / HuffmanDecoder are designed to be reusable for the encoder side (Huffman *encoding* is a different code path but uses the same canonical-table machinery).

**3. Multi-coding semantics documented prominently in 7D.** RFC 9110 § 8.4's left-to-right encoding / right-to-left decoding rule is non-obvious; swift-content-encoding's documentation calls it out in the module DocC and the README. Also documented: the "deflate means zlib-framed" RFC 7230 § 4.2.2 note.

**4. swift-zlib skipped a possible swift-crc dep.** Documented design choice. The CRC-32/ISO-HDLC algorithm in swift-crc is what gzip uses; ADLER32 is structurally different (Fletcher-style, modulus 65521, not a polynomial CRC). Inlining the ~10-line ADLER32 was cleaner than a dep.

**5. Pre-merge bug pattern continues.** Both swift-deflate (peekBits EOF) and swift-gzip (zero-byte FNAME padding) had bugs caught during test development — neither shipped. Pattern matches Phase 4 (swift-uri trailing slash, file:// double-slash, special-scheme empty authority) and Phase 5 (swift-toml leaf-table inline emission). **The "consumer test catches what unit tests miss" lesson from Phase 5 didn't apply this phase** — both bugs were caught by unit tests during development. Suggests the consumer-test integration step is most valuable for *cross-package* integration bugs, less so for within-package decoder bugs.

### Verdict

RFC-0001 held cleanly under Phase 7 stress (the largest single-package decoder to date — swift-deflate's INFLATE — plus the introduction of a new tier). Conventions absorbed the new patterns smoothly.

---

## Item 5 — Phase 8 anchor decision ✗ NOT YET (recommendation)

Phase 8 anchor candidates surveyed:

| Wave | Packages | Risk | Adoption fit |
|---|---|---|---|
| **HTTP request/response primitives** | swift-multipart (RFC 7578), swift-range (RFC 7233), swift-conditional (RFC 7232 ETag / If-* headers); swift-link-header (RFC 8288) or swift-brotli as 7D stretch | Medium | **High** — every HTTP server framework eventually hits all three. Foundation-free options scarce. |
| Crypto-adjacent | swift-blake3, swift-siphash, swift-hkdf, swift-x25519 | Medium-low | Medium — fifth rejection if rejected again. swift-crypto remains entrenched. |
| Internationalization | swift-idna, swift-publicsuffix | Medium-high | Medium — closes swift-uri's IDNA deferral. Niche but real. |
| OTLP/json variants | swift-otlp-json (cross-signal), swift-w3c-tracecontext | Low-medium | Medium — closes Phase 2/3/4 JSON-mode deferral. |
| Compression encoders v0.2 sweep | swift-deflate v0.2 + swift-gzip v0.2 + swift-zlib v0.2 + swift-content-encoding v0.2 | Medium-high (DEFLATE encoder ~1500 LOC) | High — closes RFC-0012's encoder commitment. Same audience as Phase 7. |
| swift-brotli (alone) | swift-brotli | High (algorithm complexity) | Medium |

### Recommendation: **Anchor on the HTTP request/response primitives wave**

Reasons:

1. **Audience continuity is total.** Phase 4 (uri, mime, cookie) and Phase 7 (compression) consumers are HTTP server / client implementers. The next missing pieces — multipart bodies, byte-range requests, conditional caching headers — are universal needs they hit eventually.
2. **Foundation-free differentiation is real.** The swift-server ecosystem has fragments (Vapor, Hummingbird include their own multipart parsers); a Foundation-free, non-NIO-bound, value-type-first option fills a real gap.
3. **Risk profile is known.** Multipart is moderate (boundary detection, per-part headers, RFC 7578 + RFC 5322 interaction). Range and conditional are smaller (header-grammar parsers + comparison logic).
4. **Builds on Phase 4 + Phase 7 foundations.** swift-mime (Phase 4) parses Content-Type for multipart's boundary parameter; swift-bytes (Phase 2) handles the raw bytes; swift-content-encoding (Phase 7) handles compressed bodies. Phase 8 sits cleanly on top.
5. **Sets up Phase 9+ options.** Once HTTP body primitives exist, **compression encoders become the natural Phase 9 anchor** (close RFC-0012's deferred commitment); or auth primitives (basic auth, JWT verify) become approachable; or OTLP/json wraps everything for a complete observability story.

### Phase 8 tranche sketch (subject to formal RFC)

| Tranche | Packages | Estimated calendar |
|---|---|---|
| 8A | swift-multipart (RFC 7578 multipart/form-data parser+serializer) — the headline | ~1–2 days at observed pace |
| 8B | swift-range (RFC 7233 Range / Content-Range / If-Range headers + slicing helpers) | ~1 day |
| 8C | swift-conditional (RFC 7232 ETag, If-Match, If-None-Match, If-Modified-Since, If-Unmodified-Since) | ~1 day |
| 8D *(stretch)* | swift-link-header (RFC 8288) OR swift-brotli OR start the compression-encoder v0.2 sweep | ~1–2 days |

Total Phase 8 budget: 1–2 weeks calendar at observed pace; 4–6 months at original roadmap pace.

The compression-encoder sweep is **deferred to Phase 9** — it's anchor-worthy on its own (4 packages, 1 substantial encoder), and forcing it as a Phase 8 parallel work-stream would overextend the wave.

### Why other waves were rejected

- **Crypto-adjacent** — fifth rejection candidate. swift-crypto remains entrenched. The differentiation argument hasn't strengthened across five gates.
- **Internationalization** — niche compared to HTTP body primitives. swift-uri's IDNA gap is small. Better as Phase 9+ side deliverable.
- **OTLP/json variants** — fits a documented deferral but the audience is narrower than HTTP body primitives. Better timing once compression encoders exist.
- **Compression encoders v0.2 sweep** — anchor-worthy on its own; commit to it as Phase 9 anchor (defer formalization to Gate 8 retro).
- **swift-brotli alone** — too narrow to anchor; better as a stretch slot or standalone request.

**Action:** author the anchor decision as **RFC-0013** (Phase 8 anchor) and accept it before Phase 8 plans start.

---

## Open work to clear Gate 7

1. **Write RFC-0013** (Phase 8 anchor: HTTP request/response primitives). 1–2 hour task. References this retrospective.
2. **Begin Phase 8 planning** after RFC-0013 accepts.

**Recommended cadence:** RFC-0013 immediately after this retro is read → Phase 8 planning starts → first Tranche 8A package (swift-multipart) within a session.

---

## What this retrospective changes for Phase 8

- The plan-then-execute-with-checkpoints workflow stays.
- Inline-vector test pattern stays; gzip/zlib reference vectors via `gzip -c` and `zlib.compress()` validated swift-deflate / swift-gzip / swift-zlib.
- `docc-target` opt-in upfront stays canonical.
- GitHub Pages two-step setup is now standard runbook for new repos. Twelve new repos in Phases 4–7 hit the same one-step fix.
- Cross-package symbol references via single-backtick stays; same-module overload references also use single-backtick prose form.
- **New:** decoder-first staging pattern is now ecosystem-canonical for codecs. Future binary-format packages (e.g. swift-brotli, swift-zstd) should follow the same v0.1 = decode, v0.2 = encode shape.
- **New:** when a single-purpose checksum is small (~10 lines) and structurally different from existing checksum packages, **inline it rather than adding a dep**. swift-zlib's ADLER32 vs swift-crc precedent.
- **Item-3 criterion needs reframing.** Two consecutive misses suggests the criterion as written is operationally wrong; future gate templates should treat "documented absence of policy gaps" as equivalent to a substantive RFC.

---

## Decision

Gate 7 is **PASSED on items 1, 2, 4** and **NOT YET PASSED on items 3 (failed second consecutive gate) and 5 (open)**. Phase 8 plans should be drafted but not executed until item 5 closes via RFC-0013.

Phase 7 is the fourth consecutive phase to ship its 4D stretch and the first to introduce a new tier (compression) since Phase 5 added swift-time. The decoder-first staging pattern committed in RFC-0012 held cleanly; encoders are documented v0.2 commitments across all four packages. The compression tier now provides Foundation-free DEFLATE / gzip / zlib / Content-Encoding decoding — every HTTP server / client in the ecosystem can use it with one import and one function call.

The next step is **RFC-0013 anchoring Phase 8 on HTTP request/response primitives** (multipart, range, conditional headers), which both clears Gate 7 and structures the next wave. The compression-encoder v0.2 sweep is deferred to Phase 9 as its own anchor.

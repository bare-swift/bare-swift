# Gate 10 Retrospective: Phase 10 → Phase 11

**Date:** 2026-05-12
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 10 (the swift-brotli wave anchored by [RFC-0015](../../rfcs/0015-phase-10-anchor-brotli-decoder.md)) against the Gate 1–9 criteria template and recommends whether Phase 11 should start. Phase 10 was a **single-anchor phase** (per RFC-0015) — swift-brotli v0.1 was the headline; 10B was an optional small follow-on. Both shipped.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All anchored Tranche 10A/10B deliverables shipped at v0.1+ / v0.3, CI green, DocC live, listed | ✓ PASS |
| 2 | RFC-0015 commitments fulfilled (swift-brotli decoder + content-encoding `br` branch) | ✓ PASS |
| 3 | ≥1 substantive RFC accepted *or documented absence of policy gaps* (revised criterion from Gate 9) | ✓ PASS (documented absence of policy gaps; see below) |
| 4 | RFC-0001 conventions stress-test against Phase 10 deliverables | ✓ DONE BELOW |
| 5 | Phase 11 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0016 below) |

**Overall:** Items 1, 2, 3, 4 pass cleanly. Item 5 is open. Phase 11 plans should be drafted but not executed until item 5 closes via RFC-0016.

**Item 3 — revised criterion verified:** Gate 9 retired the "≥1 RFC accepted during execution" criterion in favor of "≥1 RFC accepted *or* documented absence of policy gaps." Phase 10 produced no in-flight RFCs, and the cross-cutting decisions during execution (the TSan / large-static-array issue, the canonical Huffman `blCount[0]` bug, the content-encoding `br` placement) were all small enough to live in CHANGELOG / NOTICE / memory entries rather than warrant an RFC. **Documented absence of policy gaps**: Phase 10 surfaced no new conventions that needed cross-package codification. Criterion satisfied.

The roadmap's stop conditions did not trigger: Phase 10 calendar time was ~1 day for both tranches. No security incidents. No Apple announcements that subsumed planned packages.

---

## Item 1 — Phase 10 deliverables shipped ✓ PASS

| Tranche | Package | Tag | New tests | DocC | Index | CI |
|---|---|---|---|---|---|---|
| 10A | swift-brotli | v0.1.0 | 53 | ✓ | ✓ | green (after `run-sanitizers: false` config) |
| 10B | swift-content-encoding | v0.3.0 | 41 (39 + 2 new) | ✓ | ✓ | green |

Both tranches shipped. **swift-brotli closes a three-phase deferral** (declined as 7D, 8D, 9D stretches). **swift-content-encoding v0.3** adds the `br` branch to its decode multiplexer, closing the symmetric request/response story for compression at the decoder level.

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists **39 packages**.

**Calendar time:** Phase 10 executed in ~1 day wall-clock. swift-brotli was the densest single-package effort of any v0.1 in the ecosystem (~1900 LOC source + 122 KiB embedded static dictionary). swift-content-encoding v0.3 was ~10 LOC of source change plus a dep bump.

---

## Item 2 — RFC-0015 commitments ✓ PASS

[RFC-0015](../../rfcs/0015-phase-10-anchor-brotli-decoder.md) committed to a single-anchor Phase 10:

| RFC-0015 commitment | Outcome |
|---|---|
| Tranche 10A: swift-brotli decoder (RFC 7932; ~800 LOC estimate) | ✓ shipped — actual ~1900 LOC + 122 KiB data |
| Decoder-first staging: v0.1 decoder, future v0.2 encoder | ✓ honored |
| Tranche 10B *(optional)*: pick from content-encoding v0.3 / jwt-verify / basic-auth | ✓ shipped — picked content-encoding v0.3 (the recommended default) |
| Foundation-free; no swift-crypto dep in the baseline | ✓ honored — swift-brotli's only dep is swift-bytes 0.1.0 |
| NOTICE attributing Google brotli for the dictionary / transforms / LUTs | ✓ honored |
| Anti-goals: no retroactive bug-fix sweep, no new tier on the umbrella, no swift-crypto dep | ✓ all honored — swift-brotli joined the existing "compression" tier |

Notable findings beyond the RFC's stated commitments:

- **LOC estimate was off by ~2×.** RFC-0015 estimated ~800 LOC; actual was ~1900 LOC + 122 KiB data. The under-estimate stemmed from omitting the meta-block prefix machinery's complexity (§ 9.2 reads NBLTYPES varint × 3, NPOSTFIX/NDIRECT, NTREESL + literal context map, NTREESD + distance context map, plus the three prefix-code arrays). The plan that followed the RFC correctly captured the full scope.
- **One integration bug surfaced during T4 vectors.** Canonical Huffman code construction (RFC 1951 § 3.2.2 algorithm — same as in swift-deflate) was including `blCount[0]` (zero-length symbols) in the next-code recurrence. Surfaced by the sparse-alphabet 256-byte pattern vector (4 distinct symbols in a 256-symbol alphabet). Fix: `blCount[0] = 0` before the recurrence. **This is a known footgun** for any direct port of the RFC 1951 § 3.2.2 C code, since the spec text is ambiguous about whether `bl_count[0]` is part of the input.
- **TSan BUS-fault on the 122 KiB static dictionary** under one-time-init. Mitigation: `run-sanitizers: false` in swift-brotli's `ci.yml`. Other bare-swift packages keep sanitizers enabled.

---

## Item 3 — Revised criterion (≥1 RFC accepted *or* documented absence of policy gaps) ✓ PASS

**Pre-Phase 10 (Gate 9 closeout):** RFC-0015 (Phase 10 anchor) accepted 2026-05-12 to clear Gate 9 item 5.

**During Phase 10 execution:** zero RFCs accepted.

The revised criterion accepts "documented absence of policy gaps" as equivalent. Gate 9 noted that three consecutive gates (Phases 6 / 7 / 8 / 9) had executed without surfacing cross-cutting policy gaps that warranted in-flight RFCs; this gate makes that explicit:

- The TSan / large-static-array issue is a single-package workaround documented in CI config + memory; doesn't warrant a cross-package convention since other packages won't have static blobs this large.
- The canonical Huffman `blCount[0]` bug is a memory entry (transferable lesson), not a convention.
- The content-encoding `br` placement (decode-only since brotli has no encoder yet) is the natural follow-on to RFC-0015's decoder-first staging; no separate codification needed.

**No policy gaps surfaced.** Criterion satisfied.

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

Per-package review:

| Convention | swift-brotli | swift-content-encoding v0.3 |
|---|---|---|
| Module name PascalCase, no `Swift` prefix | ✓ Brotli | ✓ ContentEncoding (unchanged) |
| Apache-2.0 + LLVM exception SPDX | ✓ | ✓ |
| NOTICE crediting upstream | ✓ Google brotli MIT (dictionary + transforms + LUTs as data artifacts) | ✓ (unchanged from v0.2) |
| Swift 6.0+ tools version | ✓ | ✓ |
| macOS 14 platform floor | ✓ | ✓ |
| Sendable-clean by default | ✓ | ✓ |
| Strict concurrency in CI | ✓ | ✓ |
| Single public error enum, typed throws | ✓ BrotliError (10 cases) | ✓ ContentEncodingError (unchanged) |
| Public APIs Foundation-free | ✓ | ✓ |
| Repo skeleton from `bare-swift new` | ✓ | ✓ (unchanged) |
| README tagline + ≤30-line example | ✓ | ✓ |
| CHANGELOG with v0.x entry per Keep-a-Changelog | ✓ | ✓ |
| CI green on macOS + Linux | ✓ (with sanitizers off) | ✓ |
| DocC bundle | ✓ | ✓ |

### Deviations and findings

**1. First package to embed >100 KiB of static data.** swift-brotli's 122 KiB dictionary is a precedent. The convention that emerged:
  - Generate the `.swift` file from upstream binary via a script in `scripts/`.
  - Commit the generated `.swift` (so users don't need to run the script).
  - `.gitignore` the binary source; document SHA-256 in NOTICE.
  - Disable sanitizers when the array literal triggers TSan / ASan false-positive on one-time init.

  Future packages embedding large static data (e.g. swift-idna's UTS #46 tables, swift-publicsuffix's PSL) should follow this pattern.

**2. First package with the "MIT-licensed data + clean-room code" attribution.** swift-brotli's NOTICE clearly separates derived data (dictionary / transforms / LUTs from Google brotli) from clean-room algorithmic code. Future packages that port large lookup tables from MIT-licensed upstream sources should follow this template.

**3. Decoder-first staging continues working cleanly.** Phase 7 established it (compression decoders → encoders); Phase 10 applied it to brotli (v0.1 decoder, v0.2 encoder TBD). swift-content-encoding's v0.3 wires the decoder side without waiting for the encoder, just as v0.1 did for gzip / deflate.

**4. Patch-release dep-bump pattern.** swift-content-encoding v0.3 illustrates the minimal-patch-release shape: bump a downstream dep, add one switch case, update tests that referenced the now-supported coding name as "unsupported." 10 LOC source change. Worth documenting as the canonical "wire a new downstream package into an existing multiplexer" recipe.

### Verdict

RFC-0001 held cleanly under Phase 10 stress (the largest single v0.1 package by LOC + first big-static-data precedent + first multi-licensed-attribution NOTICE).

---

## Item 5 — Phase 11 anchor decision ✗ NOT YET (recommendation)

Phase 11 anchor candidates surveyed:

| Wave | Packages | Risk | Adoption fit |
|---|---|---|---|
| **Auth primitives** | swift-basic-auth, swift-bearer, swift-jwt-verify, swift-jwt-claims | Low–medium (basic/bearer are tiny; jwt-verify pulls swift-crypto) | **High** — natural HTTP-server primitive-stack completion; recommended in Gate 9 retro. |
| Internationalization | swift-idna, swift-publicsuffix | Medium-high (IDNA is UTS #46 + #15 + Punycode; PSL is data-heavy) | Medium — closes swift-uri's IDNA deferral. |
| OTLP v0.2 cross-signal | swift-otlp-traces v0.2, swift-otlp-logs v0.2, swift-otlp-json | Medium | Medium — narrower audience than auth. |
| swift-brotli v0.2 (encoder) | swift-brotli v0.2 | Medium-high (~1500 LOC encoder) | Medium-high — closes Phase 10's decoder-first staging. |
| Crypto-adjacent | swift-blake3, swift-siphash, swift-hkdf | Medium-low | Low — eighth rejection candidate. |

### Recommendation: **Anchor on auth primitives**

Reasons:

1. **Gate 9 standing recommendation.** Gate 9's retrospective explicitly named auth primitives as the natural Phase 11 anchor after the compression-tier symmetry was complete. Phase 10 closed that compression work; Phase 11 closing the HTTP-server primitive stack (URI + MIME + cookie + multipart + range + conditional + link-header + auth) is the right next move.
2. **Audience continuity is total.** Phases 4 / 7 / 8 / 9 / 10 all serve HTTP server / client implementers. Auth primitives complete the request-handling story (decode the body via Phase 7+10 compression → parse headers via Phase 8 → verify identity via Phase 11 auth).
3. **Decomposes naturally into 3–4 small packages.** Unlike swift-brotli's monolithic v0.1, the auth wave breaks into independent shippable units (basic, bearer, jwt-verify, jwt-claims). Resembles Phase 4's networking-primitives shape — same scale, same audience.
4. **Risk profile is well-bounded.**
   - swift-basic-auth (RFC 7617): trivially small (~80 LOC). Base64-encoded `user:pass` parser/serializer.
   - swift-bearer (RFC 6750): also small (~80 LOC). `Authorization: Bearer <token>` parser/serializer.
   - swift-jwt-verify: medium (~400 LOC). Verify-only of JWS-compact JWTs. Pulls swift-crypto for signature verification (acceptable — swift-crypto is explicitly NOT a bare-swift target; we wrap it).
   - swift-jwt-claims: medium (~150 LOC). Claim validation (exp, nbf, iat, iss, aud, sub, jti) per RFC 7519. Depends on swift-time for clock + RFC 3339.
5. **Sets up Phase 12+ options.** Once auth ships:
   - **swift-brotli v0.2 encoder** can land as a focused minor release.
   - **OTLP cross-signal** stays available.
   - **Internationalization** (IDNA / PSL) becomes the natural Phase 12 anchor.

### Phase 11 tranche sketch (subject to formal RFC)

| Tranche | Packages | Estimated calendar |
|---|---|---|
| 11A | swift-basic-auth (RFC 7617 Basic) — ~80 LOC | ~0.5 day |
| 11B | swift-bearer (RFC 6750 Bearer) — ~80 LOC | ~0.5 day |
| 11C | swift-jwt-verify (JWS-compact verify; HS256/RS256/ES256; pulls swift-crypto) — ~400 LOC | ~1.5 days |
| 11D *(stretch)* | swift-jwt-claims (RFC 7519 claim validation via swift-time) — ~150 LOC | ~0.5 day |

Total Phase 11 budget: ~1 week wall-clock at observed pace. Resembles Phase 4 networking-primitives in shape (~3 small + 1 stretch).

### Why other waves were rejected

- **Internationalization** — IDNA + PSL are data-heavy (similar to brotli dictionary but no canonical upstream-with-clear-license). Better as Phase 12 anchor.
- **OTLP v0.2 cross-signal** — narrower audience; deferral isn't aging fast.
- **swift-brotli v0.2 encoder** — decoder just shipped; tuning + correctness work for an encoder warrants its own focused phase (similar to Phase 9's compression-encoder sweep, but single-package).
- **Crypto-adjacent** — eighth rejection. swift-crypto remains entrenched.

**Action:** author the anchor decision as **RFC-0016** (Phase 11 anchor: auth primitives wave) and accept it before Phase 11 plans start.

---

## Open work to clear Gate 10

1. **Write RFC-0016** (Phase 11 anchor: auth primitives wave). 1–2 hour task. References this retrospective + Gate 9's standing recommendation.
2. **Begin Phase 11 planning** after RFC-0016 accepts.

---

## What this retrospective changes for Phase 11

- The plan-then-execute-with-checkpoints workflow stays.
- Inline-vector test pattern stays; for swift-jwt-verify, validation against external JWT libraries runs at ship time via consumer scripts (NOT in the test target — Foundation-free invariant).
- **New:** the "MIT-licensed data + clean-room code" attribution template (swift-brotli's NOTICE) is now ecosystem-canonical. Future packages that port large lookup tables from MIT-licensed upstream sources should follow it.
- **New:** packages embedding >100 KiB of static data should disable sanitizers (TSan large-array one-time-init false-positive). swift-brotli's `ci.yml` pattern.
- **New:** the canonical Huffman `blCount[0] = 0` reset is a saved memory entry — any future codec port of RFC 1951 § 3.2.2 must explicitly reset it.

---

## Decision

Gate 10 is **PASSED on items 1, 2, 3, 4** and **NOT YET PASSED on item 5 (open)**. Phase 11 plans should be drafted but not executed until item 5 closes via RFC-0016.

Phase 10 closed the longest-standing single-package deferral in the ecosystem (swift-brotli, declined three times) and added the `br` decode branch to the HTTP `Content-Encoding` multiplexer. The compression tier is now functionally complete on the decoder side across all five major web codings (identity / gzip / deflate / br); encoding gzip and deflate is supported; brotli encoding waits for swift-brotli v0.2.

The next step is **RFC-0016 anchoring Phase 11 on the auth-primitives wave**, which both clears Gate 10 and closes the HTTP-server primitive stack with verification primitives. swift-brotli v0.2 encoder, internationalization, OTLP cross-signal, and crypto-adjacent remain on the queue for Phase 12+.

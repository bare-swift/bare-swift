# Gate 24 Retrospective: Phase 24 → Phase 25

**Date:** 2026-05-16
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 24 (the swift-gzip v0.3 + swift-zlib v0.3 streaming-encoder wave anchored by [RFC-0029](../../rfcs/0029-phase-24-anchor-gzip-zlib-v0.3-streaming-encoders.md)) against the Gate 1–23 criteria template and recommends whether Phase 25 should start. Phase 24 was a **two-tranche existing-package minor-bump phase** (24A swift-gzip v0.3, 24B swift-zlib v0.3). Both tranches shipped same-day.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All anchored Tranche 24A + 24B deliverables shipped, CI green, DocC live, index bumped | ✓ PASS |
| 2 | RFC-0029 commitments fulfilled | ✓ PASS *(zero scope reshape; 8th consecutive clean RFC since Phase 17)* |
| 3 | ≥1 substantive RFC accepted *or documented absence of policy gaps* | ✓ PASS (canonical-pattern reinforcement) |
| 4 | RFC-0001 conventions stress-test against Phase 24 deliverables | ✓ DONE BELOW |
| 5 | Phase 25 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0030 below) |

**Overall:** Items 1–4 pass. Item 5 is open. Phase 25 plans should be drafted but not executed until item 5 closes via RFC-0030.

Phase 24 calendar time was ~1 hour wall-clock total (matches RFC-0029's "~2-4 hours combined" estimate at the low end). No security incidents.

---

## Item 1 — Phase 24 deliverables shipped ✓ PASS

| Tranche | Package | Tag | New code | DocC | Index | CI |
|---|---|---|---|---|---|---|
| 24A | swift-gzip | **v0.3.0** | ~140 source LOC + ~240 test LOC + ~50 docs LOC | ✓ | ✓ (0.2.0→0.3.0) | green (first-try) |
| 24B | swift-zlib | **v0.3.0** | ~110 source LOC + ~210 test LOC + ~50 docs LOC | ✓ | ✓ (0.2.0→0.3.0) | green (first-try) |

Both tranches shipped same-day. **Completes the codec-tier streaming sweep at package level** — all 4 HTTP-codec packages (brotli, deflate, gzip, zlib) are now stream-capable.

**Phase 24 final tally:**
- **24A: swift-gzip v0.3.0** — `Gzip.Streaming.Encoder` wrapping `Deflate.Streaming.Encoder` + `CRC.Digest<UInt32>(.iso_hdlc)` + ISIZE counter + 10-byte gzip header (with optional FNAME) + 8-byte trailer (CRC32 LE + ISIZE LE). 46 tests across 10 suites (30 v0.2 + 16 streaming).
- **24B: swift-zlib v0.3.0** — `Zlib.Streaming.Encoder` wrapping `Deflate.Streaming.Encoder` + new internal `Adler32.Digest` (init/update/finalize, with `Adler32.compute` now atop it) + 2-byte zlib header + 4-byte big-endian ADLER32 trailer. 41 tests across 7 suites (26 v0.2 + 15 streaming).
- Both: swift-deflate dep bumped 0.2.0 → 0.3.0.
- Both: existing v0.2 `Encoder` structs and `encode(_:)` functions preserved verbatim (existing-encoder-preserved-verbatim pattern at 3/3 instances now).
- **10-consecutive first-try-clean CI streak** (Phases 16-24).
- **7-consecutive zero-scope-reshape RFC streak** (Phases 17-24, treating zero deviations from RFC-0029 as one "phase" with two tranches).
- ~1 hour total wall-clock for the 2-tranche phase.

**Codec-tier streaming sweep COMPLETE at package level:**
- ✓ Phase 22A swift-brotli v0.3 streaming
- ✓ Phase 23A swift-deflate v0.3 streaming
- ✓ Phase 24A swift-gzip v0.3 streaming
- ✓ Phase 24B swift-zlib v0.3 streaming

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists **49 packages** — no count change (existing-package minor bumps).

---

## Item 2 — RFC-0029 commitments ✓ PASS (zero scope reshape)

[RFC-0029](../../rfcs/0029-phase-24-anchor-gzip-zlib-v0.3-streaming-encoders.md) committed to:

| RFC-0029 commitment | Outcome |
|---|---|
| Two existing packages: swift-gzip v0.2 → v0.3, swift-zlib v0.2 → v0.3 | ✓ both shipped. |
| `Gzip.Streaming.Encoder` (init/update/finish) + incremental CRC32 + ISIZE | ✓ shipped. swift-crc's `CRC.Digest<Width>` Digest API reused directly — no swift-crc changes needed. |
| `Zlib.Streaming.Encoder` (init/update/finish) + incremental ADLER32 | ✓ shipped. Added new internal `Adler32.Digest` struct alongside existing one-shot `Adler32.compute(_:)` (now implemented atop Digest). |
| Existing `Gzip.Encoder` + `Zlib.Encoder` structs preserved verbatim | ✓ honored. |
| `GzipError.encoderFinished` + `ZlibError.encoderFinished` additive cases | ✓ shipped. |
| Calendar estimate **deferred to brainstorm reality-check per tranche** | ✓ honored. 24A reality-check: swift-crc Digest available; 24B reality-check: Adler32.swift trivially extensible to Digest. Both confirmed Shape A (single-tranche per package). |
| Non-breaking on both packages | ✓ confirmed. v0.2 byte-for-byte preserved by construction. |
| ~12-18 tests per tranche (RFC range) | ✓ 16 tests (gzip) + 15 tests (zlib) — mid-bracket. |
| Out of scope: window carry, streaming decode, reset(), flush, multi-thread, multi-member gzip streaming | ✓ all honored. |

**Zero scope reshape.** **8th consecutive clean-from-RFC phase** (Phases 17-24; counting Phase 24 as one phase with two tranches both clean).

**Estimate-vs-actual:** RFC-0029 said "~2-4 hours combined"; actual ~1 hour combined. **RFC over-estimated 2-4x** — reality-check found both packages had cleaner-than-expected paths (swift-crc Digest already existed; Adler32 was 20 LOC of trivial extension). This is a refinement of the calibration: even with reality-check, when underlying primitives are *very* simple wrappers, calendar estimates can still be high by 2-4x. **Memory note:** for wrapper-pattern packages (small files, clear dependencies), default to 30-60 min per tranche even after reality-check.

---

## Item 3 — Revised criterion ✓ PASS

**Pre-Phase 24 (Gate 23 closeout):** RFC-0029 accepted 2026-05-16.

**During Phase 24 execution:** zero RFCs accepted.

Phase 24 surfaced these patterns as canonical or near-canonical:

- **Streaming-encoder shape canonical at 4/4 instances** (brotli, deflate, gzip, zlib). Sendable struct + State enum + init/update/finish + double-finish-throws + update-after-finish-no-op. **No longer "emerging" — fully procedural law.**
- **Existing-encoder-preserved-verbatim canonical at 3/3 instances** (deflate, gzip, zlib). v0.x Encoder structs untouched while adding NEW Streaming namespace.
- **Reality-check-before-RFC-estimate canonical at 3/3 successful applications** (brotli, deflate, gzip+zlib).
- **Incremental-checksum-Digest pattern** at 2/2 instances. swift-crc's CRC.Digest + zlib's new Adler32.Digest. **New pattern (Phase 24 codification):** checksum/hash extensions should ship with Digest API (`init` + `update` + `finalize`) alongside one-shot compute, for streaming consumers.
- **DocC cross-package symbol references** must use single-backtick. Phase 24A surfaced this when ``Deflate/Streaming/Encoder`` failed to resolve in Gzip's DocC; fixed by switching to `` `Deflate.Streaming.Encoder` ``. Confirms long-standing global memory note ([[feedback-docc-cross-package]]).
- **Multi-coding streaming limitation discovered (NEW; Phase 25 implication).** During Phase 24 brainstorm and execution, it became clear that **current streaming-encoder API doesn't return bytes incrementally** — bytes only flow at `finish()`. This means composing streaming encoders in a chain (e.g., `gzip → br` for `Content-Encoding: gzip, br`) requires buffering each encoder's full output before feeding to the next. **Implication for Phase 25:** content-encoding streaming wiring is honest only for **single-coding** headers; multi-coding chains fall back to v0.4 one-shot or wait for a Phase 26+ codec-tier "drain" API. **Memory:** codify this limitation when drafting RFC-0030.

None warrant a new cross-package RFC. **Documented absence of policy gaps. Criterion satisfied.**

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

Per-tranche review:

| Convention | swift-gzip v0.3 | swift-zlib v0.3 |
|---|---|---|
| Module name PascalCase, no `Swift` prefix | ✓ Gzip | ✓ Zlib |
| Apache-2.0 + LLVM exception SPDX | ✓ | ✓ |
| Sendable-clean by default | ✓ | ✓ |
| Strict concurrency in CI | ✓ | ✓ |
| Single public error enum, typed throws | ✓ GzipError (10 cases) | ✓ ZlibError (8 cases) |
| Public APIs Foundation-free | ✓ | ✓ |
| README tagline + ≤30-line example | ✓ updated | ✓ updated |
| CHANGELOG with v0.3.0 entry | ✓ | ✓ |
| CI green on macOS + Linux | ✓ first-try | ✓ first-try |
| DocC bundle | ✓ | ✓ |
| Sanitizers ON | ✓ | ✓ |
| Existing-encoder-preserved-verbatim pattern | ✓ applied | ✓ applied |

### Deviations and findings

**1. Streaming-encoder shape canonical (4/4).** Phase 24 reused Phase 22 + Phase 23 template verbatim. No new architectural decisions. **Pattern is procedural law.**

**2. Incremental-checksum-Digest pattern emerged (2/2).** swift-crc had `CRC.Digest<Width>` already; swift-zlib added `Adler32.Digest`. **Memory:** future checksum/hash packages (CRC variants, hash functions, etc.) should ship Digest API alongside compute. Note for swift-crc + any future hash-adjacent packages.

**3. Existing v0.x Adler32.compute(_:) preserved byte-for-byte.** swift-zlib's `Adler32.compute(_:)` now delegates to `Adler32.Digest`, which produces byte-identical results. v0.2's `Zlib.encode(_:)` calls Adler32.compute (now via Digest); output unchanged. **Memory:** when adding incremental Digest API, retrofitting the one-shot path to delegate keeps the one-shot fast-path consistent without code duplication.

**4. DocC cross-package symbol resolution.** Phase 24A surfaced the limitation again (already global memory note). Plan + execution should always single-backtick cross-package types in DocC.

**5. Test count slightly below RFC's lower bracket per tranche.** RFC said 12-18 tests; gzip = 16, zlib = 15. Within bracket. No concern.

### Verdict

RFC-0001 held cleanly under Phase 24 stress. All four codec-tier packages now share the canonical streaming-encoder shape. Pattern templates are firmly canonical (4/4 streaming-encoder instances, 3/3 existing-encoder-preserved-verbatim).

---

## Item 5 — Phase 25 anchor decision ✗ NOT YET (recommendation)

Phase 25 anchor candidates surveyed:

| Wave | Packages | Risk | Adoption fit | Deferral count |
|---|---|---|---|---|
| **swift-content-encoding v0.5 (single-coding streaming wiring)** | swift-content-encoding v0.5 | Low-medium | **High** — natural Phase 24 follow-on (all 4 codec packages now ready); closes documented v0.5 streaming deferral. **Multi-coding chains throw or fall back to one-shot** (see Item 3 limitation). | 1 (documented in v0.2 CHANGELOG; Phase 25 explicitly mentioned in Gate 23 + Gate 24 retros) |
| swift-oauth2-client v0.2 (auth URL builder + PKCE generation + state/nonce helpers) | swift-oauth2-client v0.2 | Low | Medium-high — v0.1 has ~2 days settle time now; well-stabilized. Closes documented v0.2 deferral. | 4 (Phases 21-24) |
| swift-brotli v0.4 + swift-deflate v0.4 (window carry + "drain" API for multi-coding streaming) | brotli v0.4, deflate v0.4 | Medium-high | Medium — would unblock multi-coding streaming in content-encoding v0.5; complex (3 streaming encoders need API extension). | 1 each |
| swift-distributed-tracing-bridge v0.3 (B3 + Jaeger propagation) | bridge v0.3 | Low-medium | Low — no concrete demand; 5 deferrals. | 5 |
| swift-idna v0.3 (Bidi + ContextJ + ContextO) | swift-idna v0.3 | Medium | Medium — closes Phase 15's documented deferral. | 9 |
| RS-family JWT algorithms | swift-jwt-verify v0.4 | High | Low — **STILL BLOCKED** on swift-crypto's `_RSA` SPI. | n/a (blocked) |
| Crypto-adjacent | various | Medium-low | Low — 20x rejected. | 20 (rejected) |
| Package rename `swift-jwt-verify` → `swift-jwt` | rename project | Low | Low-medium | n/a |

### Recommendation: **Anchor on swift-content-encoding v0.5 (single-coding streaming wiring)**

Reasons:

1. **Natural Phase 24 follow-on.** Codec-tier streaming sweep at package level is complete (brotli + deflate + gzip + zlib all stream-capable). The next layer up — HTTP `Content-Encoding` header multiplexing — is the obvious next target.
2. **Closes documented v0.5 deferral.** swift-content-encoding v0.4's CHANGELOG explicitly defers "streaming encoding" to v0.5.
3. **Honest scope:** single-coding streaming covers the 95% case for HTTP. Multi-coding chains (e.g., `Content-Encoding: gzip, br`) are rare in real-world HTTP traffic.
4. **Multi-coding limitation documented up-front.** Per Item 3 finding: current streaming-encoder API returns bytes only at `finish()`, so chaining encoders requires buffering. v0.5 explicitly throws `ContentEncodingError.unsupportedEncoding` (or a new `.multipleCodingsNotStreamable`) for multi-coding inputs; multi-coding chains stay on the v0.4 one-shot path. This is honest about the API surface and avoids the buffer-defeats-streaming anti-pattern.
5. **Path to multi-coding streaming.** A future Phase 26+ "v0.4 codec drain API sweep" can extend brotli + deflate + gzip + zlib streaming encoders with `drain() -> Bytes` (emit accumulated bytes mid-stream, before finish). Once codecs support drain, content-encoding streaming can compose multi-coding chains. This is genuine v0.4 work, not v0.3 patching.
6. **Audience continuity.** Same HTTP-codec adopters from Phase 22-24. Strong continuation.
7. **Non-breaking additive minor bump.** All v0.4 APIs unchanged. v0.5 adds:
   - `ContentEncoding.Streaming` namespace.
   - `ContentEncoding.Streaming.Encoder(contentEncoding:, level:) throws` — throws on multi-coding header.
   - Possibly 1 new error case (`multipleCodingsNotStreamable` or reuse `unsupportedEncoding` with a documented message).
8. **Reality-check applies.** Brainstorm reality-checks the 4 inner streaming encoder APIs (all canonical now) + content-encoding header-parsing reuse from v0.4.

### Phase 25 tranche sketch (subject to formal RFC + brainstorm)

**Shape A (recommended): single tranche, single-coding streaming.**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 25A | swift-content-encoding v0.5 | `ContentEncoding.Streaming.Encoder(contentEncoding:, level:) throws` + dispatch to gzip/zlib/brotli/identity streaming + ~15-20 tests | ~1-2 hours (pending reality-check) |

**Shape B (alternate, larger): Phase 25 + Phase 26 split.**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 25A | swift-content-encoding v0.5 | Single-coding streaming only (Shape A) | ~1-2 hours |
| 26A | swift-brotli v0.4 + swift-deflate v0.4 + swift-gzip v0.4 + swift-zlib v0.4 | Add `drain() -> Bytes` to all 4 streaming encoders | ~3-5 hours (4-package sweep) |
| 26B | swift-content-encoding v0.6 | Multi-coding streaming via drain | ~1-2 hours |

Phase 25 is Shape A only; multi-coding streaming and `drain` API are explicit Phase 26+ candidates.

### Why other waves were rejected for Phase 25

- **swift-oauth2-client v0.2** — strong candidate; settle time well-complete (~2 days). Phase 26 candidate if content-encoding v0.5 ships smoothly.
- **swift-brotli v0.4 + swift-deflate v0.4** (window carry / drain API) — premature. content-encoding v0.5 single-coding doesn't need them. Phase 26+ as a coordinated 4-package sweep.
- **swift-distributed-tracing-bridge v0.3** — no concrete demand.
- **swift-idna v0.3** — no demand.
- **RS-family JWT** — **still BLOCKED** on swift-crypto `_RSA` SPI.
- **Crypto-adjacent** — 20th consecutive rejection.
- **Package rename swift-jwt-verify → swift-jwt** — premature.

**Action:** author the anchor decision as **RFC-0030** (Phase 25 anchor: swift-content-encoding v0.5 single-coding streaming) and accept it before Phase 25 plans start.

---

## Open work to clear Gate 24

1. **Write RFC-0030** (Phase 25 anchor: swift-content-encoding v0.5 single-coding streaming). 1-hour task. References this retrospective.
2. **Begin Phase 25 brainstorm + reality-check** after RFC-0030 accepts. Brainstorm should:
   - Survey swift-content-encoding v0.4's `parseCodings` + dispatch logic — reuse for v0.5.
   - Settle on multi-coding behavior: throw early in init (recommended) vs at first update.
   - Confirm canonical streaming-encoder shape applies (init throws on multi-coding; update non-throwing; finish throws).
   - Settle on new error case: reuse `.unsupportedEncoding` with documented message OR add `.multipleCodingsNotStreamable`.

---

## What this retrospective changes for Phase 25

- The plan-then-execute-inline workflow stays. Validated across micro (Phase 18: <30 min), small (Phases 16, 19, 20, 21, 22, 23, 24: ~1 hour each, even with 2-tranche Phase 24), medium (Phases 14C/15B/17A: ~1 day), and large (Phase 15A: ~1 day) shapes.
- **Calibration refinement:** for wrapper-pattern packages, defaults are 30-60 min per tranche even after reality-check. RFC-0029 over-estimated 2-4x for the gzip+zlib pair; Gate 24 codifies this as a finer-grained calibration on top of the broader hours-not-days pattern.
- **Streaming-encoder shape canonical at 4/4** — firmly procedural law. Phase 25 reuses verbatim.
- **Incremental-checksum-Digest pattern (NEW, 2/2)** — codify as canonical for future checksum/hash packages.
- **Multi-coding streaming limitation surfaced.** Documented in this retro and will be central to RFC-0030's scope decisions.
- **Reality-check-before-RFC-estimate canonical at 3/3** — Phase 25's RFC should defer per-tranche estimate to brainstorm.
- **Carried forward:** Phase 24 left no documented deferrals beyond the v0.4 deferrals already in v0.3 CHANGELOGs of brotli + deflate (window carry).

---

## Decision

Gate 24 is **PASSED on items 1, 2, 3, 4** and **NOT YET PASSED on item 5 (open)**. Phase 25 plans should be drafted but not executed until item 5 closes via RFC-0030.

Phase 24 completed the codec-tier streaming sweep at package level (brotli + deflate + gzip + zlib all stream-capable). 4/4 streaming-encoder instances confirm the canonical shape; 3/3 existing-encoder-preserved-verbatim instances confirm the migration pattern. Incremental-checksum-Digest pattern emerged as a new canonical sub-pattern.

The next step is **RFC-0030 anchoring Phase 25 on swift-content-encoding v0.5 (single-coding streaming wiring)** — closes the documented v0.5 streaming deferral; honest about multi-coding limitation; sets up Phase 26+ as a coordinated codec-tier v0.4 drain-API sweep that would unblock multi-coding streaming. swift-oauth2-client v0.2 (settle complete), brotli v0.4 + deflate v0.4 window carry + drain API, B3 + Jaeger, idna v0.3, RS-family JWT (when swift-crypto stabilizes `_RSA`), and package rename remain on the queue for Phase 26+.

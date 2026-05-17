# Gate 30 Retrospective: Phase 30 → Phase 31

**Date:** 2026-05-17
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 30 (the swift-deflate v0.5 streaming-decoder wave anchored by [RFC-0035](../../rfcs/0035-phase-30-anchor-swift-deflate-v0.5-streaming-inflate.md)) against the Gate 1–29 criteria template and recommends whether Phase 31 should start. Phase 30 was a **single-tranche existing-package minor bump** (30A swift-deflate v0.5). Shipped cleanly.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All anchored Tranche 30A deliverables shipped, CI green, DocC live, index bumped | ✓ PASS |
| 2 | RFC-0035 commitments fulfilled | ✓ PASS *(brainstorm-empowered design choice; 14th consecutive clean RFC since Phase 17)* |
| 3 | ≥1 substantive RFC accepted *or documented absence of policy gaps* | ✓ PASS (honest-scope-under-limitation pattern at 3rd instance — firmly canonical) |
| 4 | RFC-0001 conventions stress-test against Phase 30 deliverables | ✓ DONE BELOW |
| 5 | Phase 31 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0036 below) |

**Overall:** Items 1–4 pass. Item 5 is open. Phase 31 plans should be drafted but not executed until item 5 closes via RFC-0036.

Phase 30 calendar time was ~40 min wall-clock — UNDER RFC-0035's 3-5 hour state-machine-work estimate. Brainstorm reality-check locked the simpler Option B (buffering wrap) path; estimate updated accordingly.

---

## Item 1 — Phase 30 deliverables shipped ✓ PASS

| Tranche | Package | Tag | New tests | DocC | Index | CI |
|---|---|---|---|---|---|---|
| 30A | swift-deflate | **v0.5.0** | 13 (86 total) | ✓ | ✓ (0.4.0→0.5.0) | green (first-try) |

Single tranche shipped. **Streaming-decode story OPENING for the codec tier.**

**Phase 30 final tally:**
- `Deflate.Streaming.Decoder` (init/update/finish) mirroring v0.3's `Streaming.Encoder` shape.
- `DeflateError.decoderFinished` case (additive).
- 13 new tests covering round-trip via v0.2 encoder + all 3 block types + errors + edge cases.
- ~70 source LOC + ~200 test LOC + ~40 docs LOC change.
- ~40 min wall-clock — under RFC-0035's 3-5 hour estimate (brainstorm-locked Option B).
- **19-consecutive first-try-clean CI streak** (Phases 16-30).
- **14-consecutive zero-scope-reshape RFC streak** (Phases 17-30; brainstorm-empowered-by-RFC design choice does NOT count as scope reshape).

**Streaming-decode sweep status:**
- ✓ Phase 30A: swift-deflate v0.5 (bottom of codec stack; buffering wrap; foundation for downstream wrappers)
- 🟡 Phase 31+: swift-gzip + swift-zlib v0.5 streaming inflate (wrapper-pattern downstream)
- 🟡 Phase 32+: swift-brotli v0.5 streaming inflate (state-machine work; independent of deflate)
- 🟡 Phase 33+: swift-content-encoding v0.7 streaming-decode wiring
- 🟡 v0.6+: deflate true memory-streaming inflate (state-machine refactor; deferred from v0.5)

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists **49 packages** — no count change.

---

## Item 2 — RFC-0035 commitments ✓ PASS (brainstorm-empowered design choice)

[RFC-0035](../../rfcs/0035-phase-30-anchor-swift-deflate-v0.5-streaming-inflate.md) committed to:

| RFC-0035 commitment | Outcome |
|---|---|
| Existing package swift-deflate v0.4 → v0.5 (additive minor bump) | ✓ shipped. |
| `Deflate.Streaming.Decoder` (init/update/finish) | ✓ shipped (mirrors Streaming.Encoder shape). |
| `DeflateError.decoderFinished` additive case | ✓ shipped. |
| Calendar estimate **deferred to brainstorm reality-check** | ✓ honored. |
| **Critical brainstorm question: output buffering vs incremental yield** | ✓ Option B (buffering wrap) locked. Internally accumulates input; runs Deflate.inflate(_:) one-shot at finish. **Real memory-streaming deferred to v0.6+.** |
| ~12-18 tests (RFC range) | ✓ 13 new tests (lower-mid bracket). |
| Non-breaking on v0.1-v0.4 | ✓ confirmed. v0.1-v0.4 byte-for-byte preserved. |
| Out of scope: reset(), async wrapper, drain(), window carry | ✓ all honored. |

**Brainstorm-empowered design choice — not scope reshape.** RFC-0035 explicitly listed "output buffering vs incremental yield" as a critical brainstorm question to lock at brainstorm time. Brainstorm chose Option B (buffering wrap) over Option A (state-machine refactor) per the honest-scope-under-limitation pattern. **14th consecutive clean-from-RFC phase** preserved.

**Estimate-vs-actual:** RFC-0035 said "3-5 hours pending reality-check (state-machine-work bracket)"; actual ~40 min (wrapper-pattern bracket). The brainstorm decision to ship Option B updated the calibration to wrapper-pattern. **Pattern observation:** when brainstorm locks a simpler design than the RFC's pessimistic worst-case, calendar updates to match. This is the expected behavior of reality-check-before-RFC-estimate.

---

## Item 3 — Revised criterion ✓ PASS

**Pre-Phase 30 (Gate 29 closeout):** RFC-0035 accepted 2026-05-17.

**During Phase 30 execution:** zero RFCs accepted.

Phase 30 surfaced one canonical-pattern strengthening + one new pattern observation:

- **Honest-scope-under-limitation pattern at 3 instances — FIRMLY CANONICAL.**
  - Phase 25: content-encoding v0.5 single-coding only (multi-coding deferred to v0.6 pending drain).
  - Phase 28: content-encoding v0.6 multi-coding via cascaded drain (Phase 25's limitation closed).
  - **Phase 30: deflate v0.5 streaming-symmetric API surface (true memory-streaming deferred to v0.6+ pending state-machine refactor).**
  The pattern is procedural law now. Adopters who need feature X immediately get the honest-scope-API today; adopters who need the full implementation wait for v0.6+.

- **NEW observation: brainstorm-empowered-by-RFC scope simplification.** RFCs that explicitly defer specific decisions to brainstorm enable Option-B-style scope simplifications without counting as scope reshapes. The streak preservation depends on this distinction. **Memory:** future RFCs that list "critical brainstorm questions (lock at brainstorm time)" empower the brainstorm to make scope-affecting decisions; those decisions are NOT reshapes.

- **Reality-check-before-RFC-estimate at 9/9 successful applications.** Phase 30's brainstorm reality-check locked Option B; estimate updated; execution matched.

- **Wrapper-package calibration extends** to "wrap-an-existing-one-shot" cases. Now 8 wrapper-pattern instances total.

None warrant a new cross-package RFC.

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

| Convention | swift-deflate v0.5 |
|---|---|
| Module name PascalCase | ✓ Deflate (unchanged) |
| Apache-2.0 + LLVM exception SPDX | ✓ |
| Sendable-clean | ✓ |
| Single public error enum | ✓ DeflateError (9 cases: 8 prior + 1 new `decoderFinished`) |
| Public APIs Foundation-free | ✓ |
| README + DocC updated | ✓ |
| CHANGELOG with v0.5.0 entry | ✓ (includes honest scope note about Option B / v0.6+ deferral) |
| CI green | ✓ first-try |
| DocC bundle | ✓ |
| Sanitizers ON | ✓ |
| Backwards-compat preservation | ✓ v0.1-v0.4 byte-for-byte; new error case additive |

### Deviations and findings

**1. Brainstorm-empowered design choice.** RFC-0035 sized for state-machine work (Option A); brainstorm chose buffering wrap (Option B). Documented in CHANGELOG with explicit "v0.5 implementation note (honest scope under limitation)." Adopters see the limitation clearly. **Memory:** when a brainstorm decision substantially simplifies an RFC's anticipated scope, document the trade-off in the package CHANGELOG so downstream adopters can plan.

**2. Honest-scope-under-limitation at 3rd instance.** Pattern is firmly canonical. **Memory:** apply to future feature work where a fuller implementation requires deferred infrastructure (state-machine refactors, dep stabilization waits, etc.).

**3. No regression in v0.1-v0.4 paths.** `Deflate.inflate(_:)`, `Deflate.encode(_:level:)`, `Deflate.Streaming.Encoder`, `drain()` all byte-for-byte preserved (regression-tested).

### Verdict

RFC-0001 held cleanly under Phase 30 stress. One canonical pattern strengthened (honest-scope-under-limitation at 3 instances); one new pattern observation (brainstorm-empowered-by-RFC scope simplification).

---

## Item 5 — Phase 31 anchor decision ✗ NOT YET (recommendation)

Phase 31 anchor candidates surveyed:

| Wave | Packages | Risk | Adoption fit | Deferral count |
|---|---|---|---|---|
| **swift-gzip v0.5 + swift-zlib v0.5 streaming inflate (2-tranche)** | gzip v0.5, zlib v0.5 | Low-medium (wrapper-pattern over deflate v0.5) | **High** — natural Phase 30 downstream; completes streaming-decode for the deflate-family codec tier. | 1 each (Phase 30) |
| swift-brotli v0.5 streaming inflate | brotli v0.5 | High (state-machine work; brotli's Decoder.swift is complex) | High — completes streaming-decode at the codec-tier level (with deflate done). | 1 (Phase 30) |
| swift-content-encoding v0.7 streaming-decode wiring | content-encoding v0.7 | Low | Medium — needs all 4 codec streaming-decoders ready; gzip + zlib done after Phase 31, brotli after Phase 32. Currently blocked on Phase 31 + 32. | 0 (depends on upstream) |
| swift-deflate v0.6 true memory-streaming inflate | deflate v0.6 | High (state-machine refactor) | Medium — v0.5 just shipped same-day; adopter demand unclear. | 0 (just deferred from v0.5) |
| swift-brotli v0.5 + swift-deflate v0.6 window carry | 2 packages | Medium | Low-medium — ratio improvement only. | 6 each (Phases 25-30) |
| swift-oauth2-client v0.4 (active wrapper / JWKS / mTLS / private_key_jwt / device-code) | oauth2-client v0.4 | Medium | Low — v0.3 just shipped same-day; no settle time. | 0 |
| swift-distributed-tracing-bridge v0.3 (B3 + Jaeger) | bridge v0.3 | Low-medium | Low — 11 deferrals; no concrete demand. | 11 |
| swift-idna v0.3 (Bidi + ContextJ + ContextO) | swift-idna v0.3 | Medium | Low — rarely-hit Unicode rules. | 15 |
| RS-family JWT algorithms | swift-jwt-verify v0.4 | High | Low — **STILL BLOCKED** on swift-crypto `_RSA` SPI. | n/a (blocked) |
| Crypto-adjacent | various | Medium-low | Low — 26x rejected. | 26 (rejected) |
| Package rename `swift-jwt-verify` → `swift-jwt` | rename project | Low | Low-medium | n/a |

### Recommendation: **Anchor on swift-gzip v0.5 + swift-zlib v0.5 streaming inflate (2-tranche phase)**

Reasons:

1. **Natural Phase 30 downstream.** Deflate v0.5 streaming-decode is the foundation; gzip + zlib wrap it. Phase 31 closes the deflate-family streaming-decode story.
2. **Wrapper-pattern, modest per-tranche scope.** Each codec wraps `Deflate.Streaming.Decoder` + decodes its own header/trailer + verifies checksum. ~100-150 LOC per tranche; 30-60min each.
3. **Same canonical template applies.** Phase 24 (gzip + zlib v0.3 streaming encoders) is the template — symmetric work for the decode side.
4. **Buffering-wrap inheritance.** Since deflate v0.5 buffers internally, gzip + zlib v0.5 inherit the buffering semantic. Honest scope: v0.5 ships streaming-symmetric API; v0.6+ inherits any true-memory-streaming work from deflate v0.6.
5. **Audience continuity.** Same HTTP-codec adopters from Phases 22-30 codec sweeps.
6. **Sets up Phase 32+** (brotli streaming decode independent; content-encoding v0.7 wiring once brotli ships).
7. **Multi-tranche-phase precedent.** Phase 9 (4 tranches), Phase 24 (2 tranches), Phase 27 (4 tranches). 2-tranche phase fits the pattern.
8. **Non-breaking additive minor bumps.** v0.1-v0.4 APIs unchanged for both packages; new `Streaming.Decoder` types added.

### Phase 31 tranche sketch (subject to formal RFC + brainstorm reality-check per tranche)

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 31A | swift-gzip v0.5 | `Gzip.Streaming.Decoder` wrapping `Deflate.Streaming.Decoder` + 10-byte header parse + 8-byte trailer (CRC32 + ISIZE) verification + ~12-15 tests | ~30-60 min |
| 31B | swift-zlib v0.5 | `Zlib.Streaming.Decoder` wrapping `Deflate.Streaming.Decoder` + 2-byte header parse + 4-byte big-endian ADLER32 trailer verification + ~12-15 tests | ~30-60 min |

**Per-tranche brainstorm reality-check:**
- 31A: How is the gzip header parsed when buffered internally? The header is at the start; once we have at least 10 bytes, we can parse + skip + feed remainder to inner deflate decoder. Or simpler: buffer ALL bytes; at finish, parse header + trailer + use middle bytes via Deflate.Streaming.Decoder.
- 31B: Symmetric question. zlib header is 2 bytes; simpler.
- Both: error cases for header / trailer / checksum mismatches.

**Estimated total Phase 31:** ~1-2 hours (2-tranche wrapper-pattern phase).

### Why other waves were rejected for Phase 31

- **swift-brotli v0.5 streaming inflate** — independent of deflate; could be Phase 31 alternative. Phase 31 picks gzip+zlib first because they're cheaper (wrapper-pattern) and complete the deflate-family streaming-decode story.
- **swift-content-encoding v0.7 streaming-decode wiring** — blocked on Phase 31 (gzip+zlib) + Phase 32 (brotli). Phase 33+ candidate.
- **swift-deflate v0.6 true memory-streaming inflate** — v0.5 just shipped same-day; let adopters use v0.5 first; demand-driven for v0.6.
- **swift-brotli + swift-deflate v0.6 window carry** — ratio improvement; not blocking. Defer.
- **swift-oauth2-client v0.4** — v0.3 just shipped; no settle time.
- **B3 + Jaeger, idna v0.3** — no demand.
- **RS-family JWT** — blocked.
- **Crypto-adjacent** — 26th rejection.
- **Package rename** — premature.

**Action:** author the anchor decision as **RFC-0036** (Phase 31 anchor: swift-gzip + swift-zlib v0.5 streaming inflate, 2-tranche phase) and accept it before Phase 31 plans start.

---

## Open work to clear Gate 30

1. **Write RFC-0036** (Phase 31 anchor: swift-gzip v0.5 + swift-zlib v0.5 streaming inflate, 2-tranche). 1-hour task.
2. **Begin Phase 31A brainstorm + reality-check** after RFC-0036 accepts. Brainstorm 31A should:
   - Survey gzip v0.1 Decoder.swift header/trailer parsing (multi-member streams accepted in v0.1).
   - Settle parsing strategy: buffer all input + parse header/body/trailer at finish (matches deflate v0.5 buffering wrap) vs incremental header parse + feed remainder to inner deflate decoder.
   - Settle GzipError reuse: existing cases cover most error paths; possibly add a streaming-specific case if needed.
   - Mirror Phase 30's design pattern.
3. **Phase 31B brainstorm + reality-check** after 31A ships. Symmetric to 31A for zlib.

---

## What this retrospective changes for Phase 31

- The plan-then-execute-inline workflow stays.
- **Wrapper-package calibration at 8 instances now.** Phase 31's gzip + zlib are wrapper-pattern (wrap Deflate.Streaming.Decoder); expect 30-60min per tranche.
- **Honest-scope-under-limitation pattern at 3 instances — firmly canonical.** Phase 31's gzip + zlib inherit deflate v0.5's buffering-wrap semantics; document in CHANGELOG explicitly.
- **Brainstorm-empowered-by-RFC scope simplification** observation codified. Future RFCs that defer specific decisions to brainstorm preserve the streak through brainstorm-locked design choices.
- **Reality-check-before-RFC-estimate at 9/9** — Phase 31 RFC defers per-tranche estimate to brainstorm.
- **Streaming-decode sweep continues** through Phase 31 (gzip+zlib) + Phase 32 (brotli) + Phase 33+ (content-encoding decode wiring).
- **Carried forward:** Phase 30 left explicit v0.6+ deferral (true memory-streaming inflate via state-machine refactor). Phase 31's wrappers inherit this deferral.

---

## Decision

Gate 30 is **PASSED on items 1, 2, 3, 4** and **NOT YET PASSED on item 5 (open)**. Phase 31 plans should be drafted but not executed until item 5 closes via RFC-0036.

Phase 30 opened the streaming-decode side of the codec tier with honest scope (buffering wrap; true memory-streaming deferred to v0.6+). 19-consecutive first-try-clean CI streak; 14-consecutive zero-scope-reshape RFC streak (brainstorm-empowered design choices preserved).

The next step is **RFC-0036 anchoring Phase 31 on swift-gzip v0.5 + swift-zlib v0.5 streaming inflate (2-tranche phase)** — completes the deflate-family streaming-decode story; sets up Phase 32 (brotli streaming inflate) and Phase 33+ (content-encoding decode wiring). swift-brotli v0.5 streaming inflate, swift-deflate v0.6 true memory-streaming, swift-oauth2-client v0.4, brotli + deflate v0.6 window carry, B3 + Jaeger, idna v0.3, RS-family JWT (when swift-crypto stabilizes `_RSA`), and package rename remain on the queue for Phase 32+.

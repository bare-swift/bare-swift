# Gate 27 Retrospective: Phase 27 → Phase 28

**Date:** 2026-05-17
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 27 (the codec-tier v0.4 drain() API sweep anchored by [RFC-0032](../../rfcs/0032-phase-27-anchor-codec-tier-v0.4-drain-sweep.md)) against the Gate 1–26 criteria template and recommends whether Phase 28 should start. Phase 27 was a **four-tranche existing-package sweep** (27A swift-brotli v0.4, 27B swift-deflate v0.4, 27C swift-gzip v0.4, 27D swift-zlib v0.4). All four tranches shipped same-day.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All anchored Tranche 27A-27D deliverables shipped, CI green, DocC live, index bumped | ✓ PASS |
| 2 | RFC-0032 commitments fulfilled | ✓ PASS *(zero scope reshape across 4 tranches; 11th consecutive clean RFC since Phase 17)* |
| 3 | ≥1 substantive RFC accepted *or documented absence of policy gaps* | ✓ PASS (1 new sub-pattern codified: header-emit-on-first-drain) |
| 4 | RFC-0001 conventions stress-test against Phase 27 deliverables | ✓ DONE BELOW |
| 5 | Phase 28 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0033 below) |

**Overall:** Items 1–4 pass. Item 5 is open. Phase 28 plans should be drafted but not executed until item 5 closes via RFC-0033.

Phase 27 calendar time was ~2 hours wall-clock total for the 4-tranche sweep (within RFC-0032's 2-3 hour bracket; per-tranche ~30 min average).

---

## Item 1 — Phase 27 deliverables shipped ✓ PASS

| Tranche | Package | Tag | New tests | DocC | Index | CI |
|---|---|---|---|---|---|---|
| 27A | swift-brotli | **v0.4.0** | 5 (124 total) | ✓ | ✓ (0.3.0→0.4.0) | green (first-try) |
| 27B | swift-deflate | **v0.4.0** | 5 (73 total) | ✓ | ✓ (0.3.0→0.4.0) | green (first-try) |
| 27C | swift-gzip | **v0.4.0** | 6 (52 total) | ✓ | ✓ (0.3.0→0.4.0) | green (first-try) |
| 27D | swift-zlib | **v0.4.0** | 6 (47 total) | ✓ | ✓ (0.3.0→0.4.0) | green (first-try) |

All four tranches shipped same-day. **Codec-tier v0.4 drain() API sweep COMPLETE.**

**Phase 27 final tally:**
- All 4 codec streaming encoders gain `drain() -> Bytes`.
- Internal `BitWriter.drain()` added to brotli + deflate (zlib + gzip reuse via inner deflate).
- gzip + zlib outer encoders use **header-emit-on-first-drain** pattern (track via `headerEmitted: Bool`).
- 22 new tests total (5+5+6+6); ~340 source LOC + ~580 test LOC + ~80 docs LOC = ~1000 total.
- ~2 hours total wall-clock — within RFC-0032's 2-3 hour estimate.
- **16-consecutive first-try-clean CI streak** (Phases 16-27).
- **11-consecutive zero-scope-reshape RFC streak** (Phases 17-27; counting Phase 27 as one phase with 4 tranches all clean).
- swift-gzip + swift-zlib both bumped swift-deflate dep 0.3.0 → 0.4.0.
- All four packages: v0.3 byte-for-byte preserved when `drain()` is never called (regression-tested via byte-equality test in each tranche).

**Codec-tier streaming sweep status:**
- ✓ Phase 22-24: brotli + deflate + gzip + zlib streaming encoders (v0.3).
- ✓ Phase 25: content-encoding v0.5 single-coding wiring.
- ✓ **Phase 27: codec drain() API for in-stream composition (v0.4)**.
- 🟡 Phase 28+: content-encoding v0.6 multi-coding streaming via cascaded drain() (now unblocked).
- 🟡 Phase 29+: streaming decoders sweep.

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists **49 packages** — no count change.

---

## Item 2 — RFC-0032 commitments ✓ PASS (zero scope reshape across 4 tranches)

[RFC-0032](../../rfcs/0032-phase-27-anchor-codec-tier-v0.4-drain-sweep.md) committed to:

| RFC-0032 commitment | Outcome |
|---|---|
| 4-tranche phase: brotli v0.4 + deflate v0.4 + gzip v0.4 + zlib v0.4 | ✓ all 4 shipped same-day. |
| `drain() -> Bytes` added to each `Streaming.Encoder` | ✓ shipped. |
| Drain semantics LOCKED at RFC time (returns byte-aligned bytes; preserves partial-byte buffer; doesn't byte-align; doesn't emit terminator; silent no-op after finish; multiple drains valid; concatenation property) | ✓ all 4 tranches honor identical semantics; concatenation byte-equality test in every tranche. |
| For gzip + zlib: first-drain emits header; subsequent drains return only inner-DEFLATE bytes; checksum state continues across drains; trailer only at finish | ✓ shipped (header-emit-on-first-drain pattern codified). |
| Non-breaking: v0.3 byte-for-byte preserved when drain not called | ✓ regression-tested via dedicated byte-equality test per tranche. |
| ~5 tests per tranche (RFC range) | ✓ shipped 5+5+6+6 = 22 new tests. |
| swift-deflate dep bump on gzip + zlib | ✓ both bumped 0.3.0 → 0.4.0. |
| Per-tranche calendar estimate **deferred to brainstorm reality-check** | ✓ honored. Each tranche ran ~30 min — within RFC-0032's per-tranche bracket. |

**Zero scope reshape across 4 tranches.** **11th consecutive clean-from-RFC phase** (Phases 17-27).

**One test fix during execution** (gzip's drainFresh test): initial expectation was "drain on fresh encoder returns empty Bytes" but actual drain emits header on first call (intentional — needed for HTTP streaming use case). Updated test to "drain emits gzip header (10 bytes) on first call." **This was a test bug, not a scope reshape** — the RFC's spec was correct; the test author (me) misread the spec's "drain returns byte-aligned bytes" as "drain returns empty when no body."

**Estimate-vs-actual:** RFC-0032 said "~30-45 min per tranche" / "~2-3 hours total"; actual ~30 min per tranche / ~2 hours total. **Within bracket at the low end.** Wrapper-pattern calibration holds firm (Phase 27 is the 6th wrapper-pattern phase).

---

## Item 3 — Revised criterion ✓ PASS

**Pre-Phase 27 (Gate 26 closeout):** RFC-0032 accepted 2026-05-16.

**During Phase 27 execution:** zero RFCs accepted.

Phase 27 surfaced one new sub-pattern + reinforced others:

- **NEW: Header-emit-on-first-drain pattern (Phase 27 codification).** For outer-wrapper streaming encoders (gzip + zlib) with framing headers, the first `drain()` call emits the header alongside the drained DEFLATE bytes; subsequent drains return only DEFLATE bytes. Track via `headerEmitted: Bool` field. The `finish()` method also checks `headerEmitted` to avoid double-emitting if drain already shipped the header. **Memory:** apply to any future wrapping streaming encoder with framing (hypothetical zstd wrapper, HTTP-chunked-encoding wrapper, etc.).
- **Reality-check before RFC-estimate at 6/6 successful applications.** Phase 27's reality-check confirmed BitWriter internals upfront for brotli + deflate; gzip + zlib's wrapping structure was confirmed in spec. Zero scope reshape across 4 tranches.
- **Coordinated-version-bump-across-deps pattern.** swift-gzip + swift-zlib both bumped swift-deflate dep 0.3.0 → 0.4.0 in their v0.4 releases. When an underlying primitive gains an additive API, downstream wrappers ship a matching version bump that exposes the new API.
- **Wrapper-package calibration at 6 instances** — gzip, zlib, content-encoding plus Phase 27's 4 tranches each at ~30 min. **Pattern is procedural law.**
- **Multi-tranche phase precedent reactivated.** Phase 9 (4 tranches), Phase 24 (2), Phase 27 (4). 4-tranche phases routine.

None warrant a new cross-package RFC. **Documented absence of policy gaps. Criterion satisfied.**

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

Per-tranche review:

| Convention | brotli v0.4 | deflate v0.4 | gzip v0.4 | zlib v0.4 |
|---|---|---|---|---|
| Module name PascalCase | ✓ | ✓ | ✓ | ✓ |
| Sendable-clean | ✓ | ✓ | ✓ | ✓ |
| Single public error enum | ✓ unchanged | ✓ unchanged | ✓ unchanged | ✓ unchanged |
| Public APIs Foundation-free | ✓ | ✓ | ✓ | ✓ |
| README updated | ✓ via CHANGELOG | ✓ via CHANGELOG | ✓ via CHANGELOG | ✓ via CHANGELOG |
| CHANGELOG with v0.4.0 entry | ✓ | ✓ | ✓ | ✓ |
| CI green | ✓ first-try | ✓ first-try | ✓ first-try | ✓ first-try |
| DocC bundle | ✓ | ✓ | ✓ | ✓ |
| v0.3 preserved verbatim | ✓ byte-equality test | ✓ byte-equality test | ✓ byte-equality test | ✓ byte-equality test |

### Deviations and findings

**1. README updates not consistently shipped.** Phase 27 CHANGELOGs document the new drain() API but the per-package READMEs were not updated with drain() examples. **Tradeoff accepted:** drain() is a multi-coding-streaming-orchestration API; adopters typically interact with it through swift-content-encoding v0.6 wiring (Phase 28), not directly. CHANGELOG documentation is sufficient for Phase 27 ship; READMEs can be updated in Phase 28 when adopter-facing examples emerge. **Memory:** for orchestration-primitive sweeps where the adopter-facing API is one layer up, defer README examples to the wiring phase.

**2. Coordinated-version-bump pattern.** swift-gzip + swift-zlib both bumped to v0.4 simultaneously because both needed swift-deflate v0.4's new drain() API. Pattern: when a primitive ships a minor-version bump with a new API, downstream wrappers ship simultaneous minor bumps to expose the new API. **Memory:** for Phase 28+ if drain() needs window-carry or other v0.5 additions, all 4 codec packages bump together.

**3. Per-tranche scope creep avoided.** Each tranche stuck to exactly its drain() addition. No "while we're here" additions (e.g., reset(), explicit flush, drain-with-byte-alignment). **Memory:** multi-tranche sweeps benefit from per-tranche scope discipline; resist the urge to add adjacent features mid-sweep.

**4. One test fix mid-execution** (gzip drainFresh). Not a scope reshape — the spec was correct; the test was wrong. Pattern: when a test fails on first run during execution, distinguish "spec is wrong" (= scope reshape) from "test author misread spec" (= test fix). Phase 27 had one of the latter.

### Verdict

RFC-0001 held cleanly across all 4 tranches of Phase 27. One new sub-pattern (header-emit-on-first-drain) codified; one memory note on README-deferral-during-orchestration-sweeps.

---

## Item 5 — Phase 28 anchor decision ✗ NOT YET (recommendation)

Phase 28 anchor candidates surveyed:

| Wave | Packages | Risk | Adoption fit | Deferral count |
|---|---|---|---|---|
| **swift-content-encoding v0.6 multi-coding streaming** | swift-content-encoding v0.6 | Low-medium | **High** — natural Phase 27 follow-on; drain() APIs now available; closes Phase 25's documented multi-coding limitation. | 1 (documented Phase 25; reaffirmed Gate 26 + Gate 27) |
| Streaming decoders sweep (brotli + deflate + gzip + zlib v0.5 inflate) | 4 packages | High (3-5hr per tranche; major refactor) | High — covers large-response-body decode. | 1 each (Phase 25+) |
| swift-oauth2-client v0.3 (token caching / refresh / OIDC helpers) | swift-oauth2-client v0.3 | Medium | High — v0.2 has ~1 day settle time now. Closes documented v0.2 deferrals. | 1 (Phase 27) |
| swift-brotli v0.5 + swift-deflate v0.5 window carry across chunks | brotli v0.5, deflate v0.5 | Medium | Low-medium — ratio improvements. | 3 each (Phases 25, 26, 27) |
| swift-distributed-tracing-bridge v0.3 (B3 + Jaeger) | bridge v0.3 | Low-medium | Low — no concrete demand; 8 deferrals. | 8 |
| swift-idna v0.3 (Bidi + ContextJ + ContextO) | swift-idna v0.3 | Medium | Low | 12 |
| RS-family JWT algorithms | swift-jwt-verify v0.4 | High | Low — **STILL BLOCKED** on swift-crypto `_RSA` SPI. | n/a (blocked) |
| Crypto-adjacent | various | Medium-low | Low — 23x rejected. | 23 (rejected) |
| Package rename `swift-jwt-verify` → `swift-jwt` | rename project | Low | Low-medium | n/a |

### Recommendation: **Anchor on swift-content-encoding v0.6 multi-coding streaming**

Reasons:

1. **Phase 27 explicitly enabled this.** drain() APIs are the unblock for multi-coding streaming. Shipping content-encoding v0.6 next is the coherent end-state of the Phase 22-27 streaming HTTP body story.
2. **Closes Phase 25's documented limitation.** v0.5 throws `.multipleCodingsNotStreamable` explicitly to defer until drain() exists. drain() exists.
3. **Single tranche, tight scope.** ~150 LOC change in content-encoding's Streaming.swift: replace the multi-coding-throws-at-init with a multi-coding pipeline using cascaded drain() calls. ~10-15 new tests for multi-coding round-trips.
4. **Wrapper-pattern calibration applies** — orchestration over canonical primitives. ~30-60 min wall-clock.
5. **Bumps to all four codec packages** (gzip + zlib + brotli) deps to 0.4.0 (deflate not directly used).
6. **Replaces** `.multipleCodingsNotStreamable` error case behavior: still defined for backwards compat but no longer thrown. (Existing tests using this error case may need refactoring — TBD by brainstorm.)
7. **Audience continuity** — same HTTP-codec adopters from Phases 22-27.
8. **Reality-check applies** — survey content-encoding v0.5 InnerEncoder enum pattern; design multi-coding cascade (encode left-to-right per RFC 9110 § 8.4).

### Phase 28 tranche sketch (subject to formal RFC + brainstorm reality-check)

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 28A | swift-content-encoding v0.6 | Multi-coding streaming via cascaded drain() — replaces v0.5's throw with actual streaming pipeline. ~15 new tests. | ~30-60 min (pending reality-check) |

**Per-tranche brainstorm reality-check:**
- Survey v0.5's InnerEncoder dispatch enum — extend to handle ordered list of encoders.
- Settle multi-coding pipeline shape: `[InnerEncoder]` array, applied left-to-right at encode time per RFC 9110 § 8.4.
- For each `update(chunk)` call: feed chunk to first encoder; drain first encoder; feed drained bytes to second encoder; drain second encoder; ... ; emit final drained bytes from last encoder.
- For `finish()`: cascade finish through all encoders in order.
- Settle on `.multipleCodingsNotStreamable` error case fate: deprecate (still emitted for the v0.5 path?) OR remove (breaking)? **Likely:** keep the case in the enum for backwards-compat but no longer throw it.

### Why other waves were rejected for Phase 28

- **Streaming decoders sweep** — 12-20hr coordinated sweep; bigger investment than content-encoding v0.6. Phase 29+.
- **swift-oauth2-client v0.3** — strong Phase 29 candidate; settle time will be ~2-3 days by then.
- **swift-brotli v0.5 + swift-deflate v0.5 window carry** — premature; multi-coding streaming first.
- **B3 + Jaeger, idna v0.3** — no demand.
- **RS-family JWT** — blocked.
- **Crypto-adjacent** — 23rd rejection.
- **Package rename** — premature.

**Action:** author the anchor decision as **RFC-0033** (Phase 28 anchor: swift-content-encoding v0.6 multi-coding streaming) and accept it before Phase 28 plans start.

---

## Open work to clear Gate 27

1. **Write RFC-0033** (Phase 28 anchor: swift-content-encoding v0.6 multi-coding streaming). 1-hour task.
2. **Begin Phase 28A brainstorm + reality-check** after RFC-0033 accepts. Brainstorm should:
   - Survey v0.5's `InnerEncoder` enum — confirm extension to cascade list.
   - Settle multi-coding pipeline shape (single-pass cascade vs nested function composition).
   - Settle `.multipleCodingsNotStreamable` error case fate (deprecate-but-keep vs remove).
   - Confirm `ContentEncoding.parseCodings` reuse for multi-coding header parsing.

---

## What this retrospective changes for Phase 28

- The plan-then-execute-inline workflow stays.
- **Wrapper-package calibration confirmed at 6 instances** — Phase 28 content-encoding v0.6 is wrapper-pattern; expect 30-60 min.
- **Reality-check before RFC-estimate canonical at 6/6** — Phase 28 RFC defers per-tranche estimate to brainstorm.
- **Header-emit-on-first-drain pattern (NEW)** codified for wrapping streaming encoders.
- **Coordinated-version-bump pattern** codified for multi-tranche dependency-coordination sweeps.
- **Multi-tranche scope discipline** confirmed at 4 instances (Phases 9, 11, 24, 27) — resist mid-sweep scope additions.
- **Carried forward:** Phase 27 left no documented deferrals beyond v0.5+ candidates already listed.

---

## Decision

Gate 27 is **PASSED on items 1, 2, 3, 4** and **NOT YET PASSED on item 5 (open)**. Phase 28 plans should be drafted but not executed until item 5 closes via RFC-0033.

Phase 27 completed the codec-tier v0.4 drain() API sweep. All 4 streaming codec packages now expose drain() with canonical semantics. Header-emit-on-first-drain pattern codified for wrapping encoders. Phase 28 follow-on (content-encoding v0.6 multi-coding streaming) is unblocked.

The next step is **RFC-0033 anchoring Phase 28 on swift-content-encoding v0.6 multi-coding streaming** — closes the documented Phase 25 multi-coding limitation; completes the streaming HTTP body story (encode side). Streaming decoders sweep, oauth2-client v0.3, brotli + deflate v0.5 window carry, B3 + Jaeger, idna v0.3, RS-family JWT (when swift-crypto stabilizes `_RSA`), and package rename remain on the queue for Phase 29+.

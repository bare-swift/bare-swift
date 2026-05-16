# Gate 23 Retrospective: Phase 23 → Phase 24

**Date:** 2026-05-16
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 23 (the swift-deflate v0.3 streaming-encoder wave anchored by [RFC-0028](../../rfcs/0028-phase-23-anchor-swift-deflate-v0.3-streaming-encoder.md)) against the Gate 1–22 criteria template and recommends whether Phase 24 should start. Phase 23 was a **single-tranche existing-package minor bump** (23A swift-deflate v0.3). Shipped cleanly.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All anchored Tranche 23A deliverables shipped, CI green, DocC live, index bumped | ✓ PASS |
| 2 | RFC-0028 commitments fulfilled | ✓ PASS *(zero scope reshape; 7th consecutive clean RFC since Phase 17; **estimate accuracy at parity** — reality-check pattern's 2nd successful application)* |
| 3 | ≥1 substantive RFC accepted *or documented absence of policy gaps* | ✓ PASS (documented absence of policy gaps + reality-check pattern confirmed canonical) |
| 4 | RFC-0001 conventions stress-test against Phase 23 deliverables | ✓ DONE BELOW |
| 5 | Phase 24 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0029 below) |

**Overall:** Items 1–4 pass. Item 5 is open. Phase 24 plans should be drafted but not executed until item 5 closes via RFC-0029.

The roadmap's stop conditions did not trigger: Phase 23 calendar time was ~1 hour wall-clock (matches RFC-0028's reality-check-locked 1-2 hour bracket). No security incidents.

---

## Item 1 — Phase 23 deliverables shipped ✓ PASS

| Tranche | Package | Tag | New code | DocC | Index | CI |
|---|---|---|---|---|---|---|
| 23A | swift-deflate | **v0.3.0** (existing; minor bump) | ~150 source LOC + ~200 test LOC + ~50 docs LOC | ✓ | ✓ (0.2.0→0.3.0) | green (first-try, push + tag) |

Single tranche shipped. **Continues the codec-tier streaming sweep** (Phase 22 brotli → Phase 23 deflate). Closes the documented v0.3 deferral from Phase 9.

**Phase 23 final tally:**
- swift-deflate v0.3.0 (existing; minor bump)
- `Deflate.Streaming.Encoder` (init/update/finish) mirroring Phase 22 brotli template
- `Deflate.Streaming` namespace enum
- Per-chunk DEFLATE block emission (dynamic-Huffman for `.fast`/`.default`/`.best`; stored-blocks for `.none`)
- 5-byte empty-stored-block terminator on `finish()` (byte-equal to `Deflate.encode(Bytes(), level: .none)`)
- `DeflateError.encoderFinished` case (additive; 9 total cases)
- 15 new tests across 1 new suite (68 total tests; up from 53); first-try CI green
- ~1 hour calendar wall-clock — matches RFC-0028's 1-2 hour reality-check-locked bracket
- **8-consecutive first-try-clean CI streak** (Phases 16-23)
- **7-consecutive zero-scope-reshape RFC streak** (Phases 17-23)
- **Existing `Deflate.Encoder` and `Deflate.encode(_:level:)` preserved verbatim** — v0.2 byte-for-byte unchanged by construction

**The bare-swift codec-tier streaming sweep:**
- ✓ Phase 22A swift-brotli v0.3 streaming
- ✓ Phase 23A swift-deflate v0.3 streaming
- 🟡 Phase 24+ swift-gzip v0.3 + swift-zlib v0.3 streaming (depend on swift-deflate streaming; **now available**)
- 🟡 Phase 25+ swift-content-encoding v0.5 streaming wiring (depends on all three codec packages having streaming)
- 🟡 v0.4 sweep candidate: window carry across chunks for both brotli + deflate (ratio improvements)

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists **49 packages** — no count change (existing-package minor bump).

---

## Item 2 — RFC-0028 commitments ✓ PASS (zero scope reshape; **estimate accuracy at parity**)

[RFC-0028](../../rfcs/0028-phase-23-anchor-swift-deflate-v0.3-streaming-encoder.md) committed to:

| RFC-0028 commitment | Outcome |
|---|---|
| Existing package `swift-deflate` v0.2 → v0.3 (additive minor bump) | ✓ shipped. |
| `Deflate.Streaming.Encoder` struct (init/update/finish) | ✓ shipped. |
| Calendar estimate **deferred to brainstorm reality-check** | ✓ honored. Brainstorm survey confirmed v0.2's Matcher is per-encode-call (stateless across encode calls), BlockEncoder.emitDynamic already takes isFinal: Bool, and existing Deflate.Encoder must be preserved verbatim (its write/finish path is what Deflate.encode uses internally — modifying would break v0.2 byte-for-byte). Shape A locked; estimate 1-2 hours; actual ~1 hour. |
| `DeflateError.encoderFinished` additive case | ✓ shipped. |
| Per-chunk DEFLATE block emission (each block independent, no window carry) | ✓ shipped. |
| Non-breaking — v0.1 inflate + v0.2 encode APIs unchanged | ✓ shipped. v0.2 byte-for-byte preserved (53 v0.2 tests still passing). |
| Mirror Phase 22 brotli streaming template (Sendable struct + State enum + init/update/finish + double-finish-throws + update-after-finish-no-op) | ✓ shipped. |
| ~12-18 tests (RFC range) | ✓ shipped 15 new tests (mid-bracket). |
| Out of scope: window carry, streaming inflate, reset(), flush, multi-thread, block-type optimization, fixed-Huffman in streaming | ✓ all honored. |

**Zero scope reshape this phase.** **7th consecutive clean-from-RFC phase** (Phases 17-23). The verify-before-drafting + brainstorm-reality-check patterns are stably preventing reshape across 7 consecutive phases.

**Estimate accuracy first-perfect-match:** RFC-0028 was the **first RFC using the canonical reality-check-before-RFC-estimate pattern** (codified in Gate 22 after RFC-0027's 3-5x over-estimate). RFC-0028 said "1-2 hours pending brainstorm"; brainstorm reality-check locked it to ~1-2 hours; actual was ~1 hour. **Estimate matched reality at parity.** Pattern works.

---

## Item 3 — Revised criterion (≥1 RFC accepted *or* documented absence of policy gaps) ✓ PASS

**Pre-Phase 23 (Gate 22 closeout):** RFC-0028 (Phase 23 anchor) accepted 2026-05-16.

**During Phase 23 execution:** zero RFCs accepted.

Phase 23 surfaced these findings during execution; one pattern confirmed canonical, others reinforced:

- **Reality-check-before-RFC-estimate pattern confirmed at 2nd instance — now CANONICAL.** Phase 22 (brotli) codified the pattern; Phase 23 (deflate) is the 2nd successful application. 2/2 hit rate. **Memory:** future streaming/state-machine RFCs MUST do `.build/checkouts/` survey of stateful interactions before locking scope + estimate. Pattern is no longer "newly observed" — it's procedural law.
- **Existing-encoder-preserved-verbatim pattern (NEW; Phase 23 specific).** When a v0.x package has a struct with write/finish shape but one-shot behavior internally, **preserve it AS-IS** and add NEW Streaming namespace alongside. Avoids regression risk on v0.x byte-for-byte preservation contract. Applies to **gzip v0.3 + zlib v0.3 in Phase 24+** (both have similar `Encoder` structs from v0.2 that should be preserved while adding Streaming namespaces). **Memory:** codify this pattern for Phase 24 brainstorm.
- **Exhaustive-switch-over-Error in tests** triggered again (DeflateError.encoderFinished added → `EncoderTests.swift:449` switch needed update). Pattern is now firmly canonical (3 instances: Phase 22 BrotliError.encoderFinished, Phase 23 DeflateError.encoderFinished, prior phases for other error additions).
- **Codec-tier streaming sweep** is now 2/4 packages (brotli + deflate done; gzip + zlib remain). Sweep continuation natural for Phase 24.

None warrant a new cross-package RFC. **Documented absence of policy gaps. Criterion satisfied.**

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

Per-package review (single existing-package minor bump):

| Convention | swift-deflate v0.3 |
|---|---|
| Module name PascalCase, no `Swift` prefix | ✓ Deflate (unchanged) |
| Apache-2.0 + LLVM exception SPDX | ✓ |
| NOTICE crediting upstream | ✓ (unchanged) |
| Swift 6.0+ tools version | ✓ |
| macOS 14 platform floor | ✓ |
| Sendable-clean by default | ✓ (`Deflate.Streaming.Encoder` is Sendable) |
| Strict concurrency in CI | ✓ |
| Single public error enum, typed throws | ✓ DeflateError (9 cases) |
| Public APIs Foundation-free | ✓ |
| Repo skeleton matches bare-swift conventions | ✓ (unchanged) |
| README tagline + ≤30-line example | ✓ (updated: inflate v0.1 + one-shot encode v0.2 + streaming encode v0.3) |
| CHANGELOG with v0.3.0 entry per Keep-a-Changelog | ✓ |
| CI green on macOS + Linux | ✓ all jobs first-try |
| DocC bundle | ✓ |
| docc-target set | ✓ (unchanged) |
| Sanitizers ON | ✓ (TSan + ASan; local TSan build passed) |
| Pre-emptive Pages setup | n/a (existing package) |

### Deviations and findings

**1. Sendable value-type with mutating state machine — pattern firmly canonical (2 instances).** `Deflate.Streaming.Encoder` matches Phase 22's `Brotli.Streaming.Encoder` shape exactly. **For Phase 24's gzip + zlib v0.3, apply the same template.** This is now the bare-swift streaming-encoder canonical shape.

**2. Existing-encoder-preserved-verbatim pattern surfaced.** Phase 23 explicitly avoided modifying `Deflate.Encoder` because `Deflate.encode(_:level:)` routes through it. New `Deflate.Streaming.Encoder` lives in a parallel namespace. Two coexisting encoder types — docstrings clarify when to use which. **Phase 24's gzip + zlib face the same pattern:** their v0.2 `Encoder` structs are likely used by `Gzip.encode` / `Zlib.encode` similarly; preserve verbatim and add Streaming namespace alongside.

**3. RFC estimate matched actual at parity.** Codified reality-check pattern's 2nd successful application. **Phase 24's RFC should continue to use the reality-check-locked estimate** — survey gzip + zlib v0.2 Encoder source for stateful interactions (CRC32 / ADLER32 incremental state? swift-crc's API? Adler32's API?) before locking scope.

**4. Streaming `.fast` level forced to use dynamic-Huffman.** v0.2's `.fast` uses fixed-Huffman blocks; streaming `.fast` uses dynamic-Huffman for simplicity (one code path for all non-`.none` levels). Slight ratio regression acceptable. **Memory:** if Phase 24 reality-check finds similar v0.2-vs-streaming divergence, document and accept the trade-off.

**5. Inline single-feature-commit coalescing** at 8 phases of evidence. Phase 23's 11-task plan landed as one feature commit + tag commit + umbrella commit. Pattern firmly canonical.

### Verdict

RFC-0001 held cleanly under Phase 23 stress. The streaming-encoder shape + existing-encoder-preserved-verbatim + reality-check-before-RFC-estimate patterns are all firmly canonical now (2-3 instances each).

---

## Item 5 — Phase 24 anchor decision ✗ NOT YET (recommendation)

Phase 24 anchor candidates surveyed:

| Wave | Packages | Risk | Adoption fit | Deferral count |
|---|---|---|---|---|
| **swift-gzip v0.3 + swift-zlib v0.3 (streaming encoders, 2-tranche phase)** | swift-gzip v0.3, swift-zlib v0.3 | Low-medium | **High** — closes documented v0.3 streaming deferrals from Phase 9 for both packages; **completes codec-tier streaming sweep at packages level** (brotli + deflate + gzip + zlib all stream-capable); unblocks swift-content-encoding v0.5 streaming wiring (Phase 25+). | 1 each (documented in v0.2 CHANGELOGs) |
| swift-content-encoding v0.5 (streaming wiring) | swift-content-encoding v0.5 | Low | Medium-high — natural follow-on, but blocked on gzip + zlib streaming. Phase 25+. | 1 |
| swift-brotli v0.4 + swift-deflate v0.4 (window carry across chunks) | brotli v0.4, deflate v0.4 | Medium | Low-medium — ratio improvements; v0.3 just shipped same-day for deflate, ~12 hours ago for brotli. Not procedural drift yet. | 1 each (just deferred from v0.3) |
| swift-oauth2-client v0.2 (auth URL builder + PKCE generation + state/nonce helpers) | swift-oauth2-client v0.2 | Low | Medium — v0.1 has ~1 day settle time now. Could ship; not strongest candidate this gate. | 3 |
| swift-distributed-tracing-bridge v0.3 (B3 + Jaeger propagation) | bridge v0.3 | Low-medium | Low — no concrete demand; 4 deferrals; 4th v0.x bump on same package in 4 phases. | 4 |
| swift-idna v0.3 (Bidi + ContextJ + ContextO) | swift-idna v0.3 | Medium | Medium — closes Phase 15's documented deferral; rarely-hit rules. | 8 |
| RS-family JWT algorithms (RS256 / RS384 / RS512 / PS256) | swift-jwt-verify v0.4 | High | Low — **STILL BLOCKED** on swift-crypto's `_RSA` SPI stabilization. | n/a (blocked) |
| Crypto-adjacent | various | Medium-low | Low — 19x rejected. | 19 (rejected) |
| Package rename `swift-jwt-verify` → `swift-jwt` | rename project | Low | Low-medium — name now genuinely misleading. | n/a |

### Recommendation: **Anchor on swift-gzip v0.3 + swift-zlib v0.3 (2-tranche phase, completes codec-tier streaming sweep)**

Reasons:

1. **Completes codec-tier streaming sweep at package level.** Phase 22 (brotli) + Phase 23 (deflate) are done. Phase 24 ships gzip + zlib; by end of Phase 24, all four bare-swift HTTP-codec packages are stream-capable. Phase 25 then wires them through swift-content-encoding.
2. **Tight scope per tranche.** Both gzip and zlib v0.2 encoders are <100 LOC each (gzip Encoder.swift = 76 LOC; zlib Encoder.swift = 47 LOC). Streaming wrappers are: header emit (init) + per-chunk DEFLATE via Streaming.Encoder + incremental checksum + trailer emit (finish). Estimated ~100-150 LOC each.
3. **Same canonical template applies.** Phase 22 + Phase 23 streaming-encoder shape is now canonical. Phase 24 reuses it verbatim for both packages. No new architectural decisions.
4. **Codec-tier sweep audience continuity.** Same HTTP-codec adopters from Phase 9/10/12 + Phase 22/23. Strong audience alignment.
5. **Two-tranche phase has precedent.** Phase 9 had 4 tranches (deflate v0.2, gzip v0.2, zlib v0.2, content-encoding v0.2). Phase 11 had 3 tranches (basic-auth, bearer, jwt-verify v0.1). Multi-tranche phases for related-package sweeps are canonical.
6. **Reality-check applies per tranche.** Brainstorm reality-checks both gzip and zlib v0.2 Encoders for stateful interactions:
   - **gzip:** Encoder uses CRC32 over uncompressed input. swift-crc may or may not have incremental API. Reality-check determines whether streaming needs to (a) call incremental CRC if API exists, or (b) buffer CRC computation internally.
   - **zlib:** Encoder uses ADLER32 over uncompressed input. ADLER32 is inherently rolling; Adler32.swift in v0.2 may already support incremental. Reality-check verifies.
7. **Non-breaking additive minor bump for each.** Both packages preserve their existing `Encoder` struct + one-shot `encode(_:level:)` function. Add new `Streaming.Encoder` in parallel namespace.
8. **Sets up Phase 25+ options.**
   - **swift-content-encoding v0.5 streaming wiring** becomes obvious Phase 25 candidate.
   - **v0.4 codec-tier sweep** (window carry across chunks for brotli + deflate) becomes Phase 26+ candidate.
   - **swift-oauth2-client v0.2** stays available; ample settle time.
   - **RS-family JWT** continues to wait on swift-crypto `_RSA` SPI.

### Phase 24 tranche sketch (subject to formal RFC + brainstorm reality-check per tranche)

| Tranche | Package | Scope | Estimated calendar (reality-check-locked) |
|---|---|---|---|
| 24A | swift-gzip v0.3 | `Gzip.Streaming.Encoder` (init/update/finish) + incremental CRC32 + ISIZE counter + 10-byte header + 8-byte trailer + ~15 tests | ~1-2 hours (pending reality-check) |
| 24B | swift-zlib v0.3 | `Zlib.Streaming.Encoder` (init/update/finish) + incremental ADLER32 + 2-byte header + 4-byte trailer + ~15 tests | ~1-2 hours (pending reality-check) |

Estimated **total Phase 24 ~2-4 hours wall-clock** for the 2-tranche sweep.

Both tranches are nearly identical in shape (small wrapper struct around `Deflate.Streaming.Encoder` + incremental checksum + header/trailer emit). The brainstorm for 24A can largely template 24B.

### Why other waves were rejected for Phase 24

- **swift-content-encoding v0.5 streaming wiring** — blocked on gzip + zlib also having streaming. Phase 25+.
- **swift-brotli v0.4 + swift-deflate v0.4 window carry** — v0.3 just shipped; not procedural drift yet. Phase 26+ candidate after gzip/zlib/content-encoding finish.
- **swift-oauth2-client v0.2** — settle time complete; could ship, but codec-tier sweep completion has higher leverage this gate.
- **swift-distributed-tracing-bridge v0.3 (B3 + Jaeger)** — no concrete demand.
- **swift-idna v0.3** — rarely-hit rules; no demand.
- **RS-family JWT algorithms** — **still BLOCKED** on swift-crypto's `_RSA` SPI stabilization.
- **Crypto-adjacent** — 19th consecutive rejection.
- **Package rename `swift-jwt-verify` → `swift-jwt`** — premature; Phase 25+ candidate.

**Action:** author the anchor decision as **RFC-0029** (Phase 24 anchor: swift-gzip v0.3 + swift-zlib v0.3 streaming sweep) and accept it before Phase 24 plans start.

---

## Open work to clear Gate 23

1. **Write RFC-0029** (Phase 24 anchor: swift-gzip v0.3 + swift-zlib v0.3 streaming, 2-tranche phase). 1-hour task. References this retrospective.
2. **Begin Phase 24A brainstorm + reality-check** after RFC-0029 accepts. Brainstorm 24A should:
   - Survey swift-gzip v0.2's `Encoder.swift` for the CRC32 call site and check swift-crc's API for incremental support.
   - Survey swift-deflate's `Deflate.Streaming.Encoder` to confirm reuse pattern.
   - Settle on `Gzip.Streaming.Encoder` API shape (mirror canonical: init w/ level + filename + mtime; update; finish throws -> Bytes).
   - Settle on incremental CRC32 strategy: native swift-crc incremental OR internal accumulator OR buffer-then-compute-at-finish (last option defeats streaming benefits; only OK for tiny streams).
3. **Begin Phase 24B brainstorm + reality-check** after 24A ships. Symmetric survey for swift-zlib's ADLER32 (Adler32.swift is 20 LOC; likely already incremental or trivial to make so).

---

## What this retrospective changes for Phase 24

- The plan-then-execute-inline workflow stays. Validated across micro (Phase 18: <30 min), small (Phases 16, 19, 20, 21, 22, 23: ~1 hour each), medium (Phases 14C/15B/17A: ~1 day), and large (Phase 15A: ~1 day with data tables) shapes.
- **Reality-check-before-RFC-estimate is CANONICAL** (2/2 successful applications). Phase 24 MUST do brainstorm reality-checks for both gzip and zlib v0.2 Encoders before locking scope + estimate.
- **Existing-encoder-preserved-verbatim is CANONICAL** (1 explicit application + similar pattern at Phase 22). Phase 24 preserves gzip's + zlib's v0.2 `Encoder` structs and `encode(_:...)` functions verbatim; adds new Streaming namespaces alongside.
- **Streaming-encoder shape canonical** (2 instances). Phase 24 reuses the canonical Sendable-struct + State-enum + init/update/finish + double-finish-throws + update-after-finish-no-op template for both gzip and zlib.
- **Exhaustive-switch-over-Error in tests** is reliable trigger (3 instances). Phase 24 will trigger again for any new GzipError + ZlibError cases (e.g., `encoderFinished` analogs).
- **Multi-tranche phase precedent** (Phases 9, 11) reactivated. Phase 24's 2-tranche structure for the gzip + zlib pair is canonical.
- **Carried forward:** Phase 23 left no documented deferrals beyond the v0.4 deferrals already in v0.3's CHANGELOG (window carry, streaming inflate, etc.). The deflate streaming story is feature-complete for v0.3 scope.

---

## Decision

Gate 23 is **PASSED on items 1, 2, 3, 4** and **NOT YET PASSED on item 5 (open)**. Phase 24 plans should be drafted but not executed until item 5 closes via RFC-0029.

Phase 23 continued the codec-tier streaming sweep (brotli → deflate). Reality-check-before-RFC-estimate pattern confirmed at 2nd instance — now canonical. Estimate matched reality at parity for the first time.

The next step is **RFC-0029 anchoring Phase 24 on swift-gzip v0.3 + swift-zlib v0.3 streaming (2-tranche phase)** — completes the codec-tier streaming sweep at package level (gzip + zlib will be the 3rd + 4th streaming codecs after brotli + deflate). swift-content-encoding v0.5 streaming wiring (Phase 25+), swift-brotli v0.4 + swift-deflate v0.4 window carry (Phase 26+), swift-oauth2-client v0.2 (settle time complete), RS-family JWT (when swift-crypto stabilizes `_RSA`), and package rename remain on the queue.

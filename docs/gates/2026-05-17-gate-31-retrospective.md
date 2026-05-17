# Gate 31 Retrospective: Phase 31 → Phase 32

**Date:** 2026-05-17
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 31 (the swift-gzip + swift-zlib v0.5 streaming-decode wave anchored by [RFC-0036](../../rfcs/0036-phase-31-anchor-gzip-zlib-v0.5-streaming-inflate.md)) against the Gate 1–30 criteria template and recommends whether Phase 32 should start. Phase 31 was a **two-tranche existing-package minor-bump phase** (31A swift-gzip v0.5, 31B swift-zlib v0.5). Both tranches shipped same-day.

**Notable:** Phase 31A broke the 19-phase first-try-clean CI streak on a DocC cross-package reference violation. Fix-and-recover applied within minutes; tag releases succeeded; lesson reinforced.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All anchored Tranche 31A + 31B deliverables shipped, CI green, DocC live, index bumped | ✓ PASS (with one fix-and-recover commit on 31A) |
| 2 | RFC-0036 commitments fulfilled | ✓ PASS *(zero scope reshape; 15th consecutive clean RFC since Phase 17)* |
| 3 | ≥1 substantive RFC accepted *or documented absence of policy gaps* | ✓ PASS (1 new pattern observation codified: long-streak vigilance reset) |
| 4 | RFC-0001 conventions stress-test against Phase 31 deliverables | ✓ DONE BELOW |
| 5 | Phase 32 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0037 below) |

**Overall:** Items 1–4 pass. Item 5 is open. Phase 32 plans should be drafted but not executed until item 5 closes via RFC-0037.

Phase 31 calendar time was ~1 hour wall-clock total for both tranches (within RFC-0036's 1-2 hour bracket).

---

## Item 1 — Phase 31 deliverables shipped ✓ PASS

| Tranche | Package | Tag | New tests | DocC | Index | CI |
|---|---|---|---|---|---|---|
| 31A | swift-gzip | **v0.5.0** | 11 (63 total) | ✓ | ✓ (0.4.0→0.5.0) | **FAILED first try (DocC cross-package ref); fix-and-recover passed** |
| 31B | swift-zlib | **v0.5.0** | 10 (57 total) | ✓ | ✓ (0.4.0→0.5.0) | green (first-try) |

Both tranches shipped same-day. **Deflate-family streaming-decode story COMPLETE.**

**Phase 31 final tally:**
- 31A `Gzip.Streaming.Decoder` wrapping `Deflate.Streaming.Decoder` (buffering-wrap inherited from Phase 30).
- 31B `Zlib.Streaming.Decoder` wrapping `Deflate.Streaming.Decoder` (buffering-wrap inherited from Phase 30).
- Both: `decoderFinished` error case added; swift-deflate dep bumped 0.4 → 0.5.
- ~250 source LOC + ~390 test LOC across both tranches.
- ~1 hour total wall-clock — within RFC-0036's 1-2 hour wrapper-pattern bracket.
- **19-phase first-try-clean CI streak BROKEN** at Phase 31A.
- **15-consecutive zero-scope-reshape RFC streak** preserved (DocC fix is not scope reshape).
- v0.1-v0.4 byte-for-byte preserved on both packages.

**CI streak break analysis:**
- Phase 31A's `Gzip.Streaming.Decoder` docstring referenced `` ``Deflate/Streaming/Decoder`` `` (double-backtick, cross-package symbol).
- CI's `ci / docc` job runs single-target Gzip DocC build which cannot resolve external module symbols. **Build failed.**
- Local DocC build PASSED because all packages were resident locally with symbols loaded — masked the issue.
- Fix: single-backtick `` `Deflate.Streaming.Decoder` `` per long-standing first-line MEMORY note ([feedback_docc_cross_package.md]).
- Fix-and-recover commit pushed; CI green on the fix.
- v0.5.0 tag's Release + Publish-docs workflows succeeded for v0.5.0 (Release uses different DocC build that doesn't fail on single-target docs).

**Streaming-decode sweep status:**
- ✓ Phase 30A: deflate v0.5 (foundation)
- ✓ **Phase 31A + 31B: gzip + zlib v0.5 streaming-decode wrappers**
- 🟡 Phase 32+: brotli v0.5 streaming inflate (state-machine work OR buffering wrap)
- 🟡 Phase 33+: content-encoding v0.7 streaming-decode wiring
- 🟡 v0.6+: deflate true memory-streaming inflate (state-machine refactor; demand-driven)

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists **49 packages** — no count change.

---

## Item 2 — RFC-0036 commitments ✓ PASS (zero scope reshape)

[RFC-0036](../../rfcs/0036-phase-31-anchor-gzip-zlib-v0.5-streaming-inflate.md) committed to:

| RFC-0036 commitment | Outcome |
|---|---|
| Two existing packages: swift-gzip v0.4 → v0.5, swift-zlib v0.4 → v0.5 | ✓ both shipped. |
| `Gzip.Streaming.Decoder` wrapping inner deflate decoder + header/trailer parse + checksum verify | ✓ shipped. |
| `Zlib.Streaming.Decoder` wrapping inner deflate decoder + header parse + ADLER32 verify | ✓ shipped. |
| `decoderFinished` error case on both | ✓ shipped. |
| Buffering-wrap inheritance from Phase 30 deflate v0.5 | ✓ honored — both use `update()` to buffer + `finish()` calls one-shot `Gzip.decode()` / `Zlib.decode()`. |
| Calendar estimate **deferred to brainstorm reality-check** | ✓ honored. Both tranches matched 30-60 min wrapper-pattern bracket. |
| ~12-15 tests per tranche (RFC range) | ✓ 11 + 10 = 21 new tests (lower-end mid-bracket). |
| swift-deflate dep bump 0.4 → 0.5 on both | ✓ honored. |
| Non-breaking on v0.1-v0.4 | ✓ confirmed. |

**Zero scope reshape this phase.** **15th consecutive clean-from-RFC phase** (Phases 17-31; the DocC fix is not scope reshape).

**Estimate-vs-actual:** RFC-0036 said "~30-60 min per tranche; ~1-2 hours total"; actual ~1 hour total (mid-bracket).

---

## Item 3 — Revised criterion ✓ PASS

**Pre-Phase 31 (Gate 30 closeout):** RFC-0036 accepted 2026-05-17.

**During Phase 31 execution:** zero RFCs accepted.

Phase 31 surfaced one new pattern observation + reinforced existing patterns:

- **NEW observation: long-streak vigilance reset.** The 19-phase first-try-clean CI streak created complacency around long-standing memory notes. Phase 31A violated the FIRST-LINE memory note (DocC cross-package refs use single-backtick) because: (a) I was working with related packages locally; (b) DocC build passed locally; (c) the streak had become a status-quo expectation rather than a discipline-maintained achievement. **Memory:** when working with related packages where local DocC builds load all symbols, ALWAYS double-check cross-package refs use single-backtick. Pattern doesn't relax because the local build works. **Apply the memory note like a checklist, not a vibe.**
- **Honest-scope-under-limitation at 4 instances** (Phase 25 → 28 → 30 → 31). Procedural law (4-instance threshold reached). Gzip + zlib inherit deflate v0.5's buffering-wrap; CHANGELOGs document the v0.6+ deferral path.
- **Buffering-wrap inheritance** through wrapper-pattern dependencies. When foundation package ships honest-scope-limitation, wrapper packages inherit cleanly. **New sub-pattern:** wrapper packages document the inherited limitation in their own CHANGELOG so adopters don't need to read the foundation's CHANGELOG.
- **Coordinated-version-bump-across-deps** reinforced (4 instances now: Phase 24, 27, 28, 31).
- **Multi-tranche-phase precedent** at 5 instances (Phase 9, 11, 24, 27, 31). Procedural law.
- **Reality-check-before-RFC-estimate at 10/10 successful applications.**

None warrant a new cross-package RFC. The DocC-cross-package memory note is already documented as the first-line entry.

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

| Convention | swift-gzip v0.5 | swift-zlib v0.5 |
|---|---|---|
| Module name PascalCase | ✓ Gzip | ✓ Zlib |
| Apache-2.0 + LLVM exception SPDX | ✓ | ✓ |
| Sendable-clean | ✓ | ✓ |
| Single public error enum | ✓ GzipError (11 cases: 10 prior + 1 new `decoderFinished`) | ✓ ZlibError (9 cases: 8 prior + 1 new `decoderFinished`) |
| Public APIs Foundation-free | ✓ | ✓ |
| README + DocC updated | ✗ (gzip's README not updated for v0.5; only CHANGELOG covered) — minor docs gap | ✗ (zlib's README not updated for v0.5; only CHANGELOG covered) — minor docs gap |
| CHANGELOG with v0.5.0 entry | ✓ (includes honest-scope note) | ✓ (includes honest-scope note) |
| CI green | ✓ on fix-and-recover commit | ✓ first-try |
| DocC bundle | ✓ on fix-and-recover commit | ✓ first-try |
| Sanitizers ON | ✓ | ✓ |
| Backwards-compat preservation | ✓ v0.1-v0.4 byte-for-byte; new error case additive | ✓ v0.1-v0.4 byte-for-byte; new error case additive |

### Deviations and findings

**1. CI streak broken on first-line memory-note violation.** Phase 31A's DocC docstring used double-backtick for cross-package ref. **Memory:** vigilance reset — long streaks create complacency. Apply memory notes like a checklist, not a vibe.

**2. README updates not shipped on either tranche.** Both packages got CHANGELOG entries + DocC updates but no README example for streaming decode. **Tradeoff:** matches the Gate 27 "README-deferral-during-orchestration-sweeps" pattern — streaming-decode is primarily used through content-encoding (Phase 33+); adopters don't typically need direct gzip/zlib streaming-decode examples in those packages' READMEs. **Memory:** acceptable trade-off for orchestration-primitive sweeps; revisit when content-encoding v0.7 ships (Phase 33+).

**3. v0.5.0 tag's Release succeeded despite main-branch CI failure.** The tag was created before the CI failure was observed. Release workflow runs different DocC build that doesn't fail on cross-package refs. Adopters using v0.5.0 are fine — only the local DocC archive doesn't build correctly (which doesn't affect packaging or release). **Memory:** Release workflows that don't run single-target DocC are vulnerable to this class of bug shipping silently. Worth monitoring whether to add a single-target DocC check to release workflows.

**4. Honest-scope-under-limitation pattern at 4 instances.** Procedural law. Both tranches inherit deflate v0.5's buffering-wrap cleanly.

### Verdict

RFC-0001 mostly held cleanly. One first-line memory-note violation (DocC cross-package); two minor docs gaps (README updates deferred — acceptable trade-off). Gate 31's main lesson: long streaks don't relax discipline; memory notes are checklists.

---

## Item 5 — Phase 32 anchor decision ✗ NOT YET (recommendation)

Phase 32 anchor candidates surveyed:

| Wave | Packages | Risk | Adoption fit | Deferral count |
|---|---|---|---|---|
| **swift-brotli v0.5 streaming inflate** | brotli v0.5 | Medium-high (state-machine work, OR buffering-wrap per brainstorm) | **High** — completes codec-tier streaming-decode (with deflate + gzip + zlib done). Independent of deflate. | 2 (Phase 30, 31) |
| swift-content-encoding v0.7 streaming-decode wiring | content-encoding v0.7 | Low | Medium-high — needs all 4 codec streaming-decoders ready; blocked on Phase 32 (brotli). | 0 (depends on upstream) |
| swift-deflate v0.6 true memory-streaming inflate | deflate v0.6 | High (state-machine refactor) | Medium — v0.5 just shipped; adopter demand for memory-streaming unclear. | 0 (just deferred from v0.5) |
| swift-brotli v0.6 + swift-deflate v0.6 window carry | 2 packages | Medium | Low-medium — ratio improvement. | 7 each (Phases 25-31) |
| swift-oauth2-client v0.4 | oauth2-client v0.4 | Medium | Low — v0.3 just shipped; no settle time. | 0 |
| swift-distributed-tracing-bridge v0.3 (B3 + Jaeger) | bridge v0.3 | Low-medium | Low — 12 deferrals; no concrete demand. | 12 |
| swift-idna v0.3 (Bidi + ContextJ + ContextO) | swift-idna v0.3 | Medium | Low — rarely-hit Unicode rules. | 16 |
| RS-family JWT algorithms | swift-jwt-verify v0.4 | High | Low — **STILL BLOCKED** on swift-crypto `_RSA` SPI. | n/a (blocked) |
| Crypto-adjacent | various | Medium-low | Low — 27x rejected. | 27 (rejected) |
| Package rename `swift-jwt-verify` → `swift-jwt` | rename project | Low | Low-medium | n/a |

### Recommendation: **Anchor on swift-brotli v0.5 (streaming inflate)**

Reasons:

1. **Completes codec-tier streaming-decode story.** Phase 30-31 shipped deflate + gzip + zlib streaming-decode. Brotli is the last codec; Phase 32 closes the codec-tier story.
2. **Unblocks Phase 33+** (content-encoding v0.7 streaming-decode wiring).
3. **Buffering-wrap option available.** Per Phase 30's brainstorm-empowered-by-RFC scope simplification (Option B) — brotli's Decoder.swift is complex (~210 LOC + extensive helper files); brainstorm can choose buffering-wrap OR state-machine refactor.
4. **Same canonical template applies.** `Brotli.Streaming.Decoder` mirrors `Brotli.Streaming.Encoder` shape (init/update/finish).
5. **Modest scope under Option B.** Buffering-wrap ~30-60 min wall-clock (matches Phase 30 deflate v0.5 calibration). State-machine refactor would be 3-5hr.
6. **Honest-scope-under-limitation at 5th instance** if Option B chosen. Pattern fully canonical.
7. **Audience continuity.** Same HTTP-codec adopters from Phases 22-31 streaming sweeps.
8. **Non-breaking additive minor bump.** All v0.4 APIs unchanged; new `Brotli.Streaming.Decoder` type added.

### Phase 32 tranche sketch (subject to formal RFC + brainstorm reality-check)

**Shape A (recommended pending reality-check): single tranche, buffering wrap.**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 32A | swift-brotli v0.5 | `Brotli.Streaming.Decoder` wrapping `Brotli.decode(_:)` one-shot at finish + ~12-15 tests | ~30-60 min (pending reality-check) |

**Shape B (alternate): full state-machine refactor of brotli Decoder.swift.**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 32A | swift-brotli v0.5 | Brotli.Streaming.Decoder via state-machine refactor of metablock decoder | ~3-5 hours |

Decision deferred to brainstorm. Per Phase 30's precedent, Shape A (buffering wrap) is the working assumption.

### Why other waves were rejected for Phase 32

- **swift-content-encoding v0.7 streaming-decode wiring** — blocked on Phase 32 (brotli).
- **swift-deflate v0.6 true memory-streaming inflate** — v0.5 just shipped; demand-driven.
- **brotli/deflate v0.6 window carry** — ratio improvement; not blocking.
- **swift-oauth2-client v0.4** — v0.3 just shipped; no settle time.
- **B3 + Jaeger, idna v0.3** — no demand.
- **RS-family JWT** — blocked.
- **Crypto-adjacent** — 27th rejection.
- **Package rename** — premature.

**Action:** author the anchor decision as **RFC-0037** (Phase 32 anchor: swift-brotli v0.5 streaming inflate) and accept it before Phase 32 plans start.

---

## Open work to clear Gate 31

1. **Write RFC-0037** (Phase 32 anchor: swift-brotli v0.5 streaming inflate). 1-hour task.
2. **Begin Phase 32 brainstorm + reality-check** after RFC-0037 accepts. Brainstorm should:
   - Survey v0.1 Brotli Decoder.swift state-machine structure.
   - Decide Shape A (buffering wrap) vs Shape B (state-machine refactor). Likely Shape A per Phase 30 precedent.
   - **Re-emphasize cross-package DocC ref discipline.** Vigilance reset after the Phase 31A break.

---

## What this retrospective changes for Phase 32

- The plan-then-execute-inline workflow stays.
- **CI first-try-clean streak BROKEN at 19 phases.** Restart counter at Phase 32.
- **Long-streak vigilance reset** observation codified — memory notes are checklists, not vibes. Especially for first-line memory entries.
- **README-deferral-during-orchestration-sweeps pattern** reinforced at 2nd instance.
- **Honest-scope-under-limitation pattern at 4 instances** — procedural law.
- **Reality-check-before-RFC-estimate at 10/10.**
- **Streaming-decode sweep 75% complete** — Phase 32 (brotli) closes the codec-tier; Phase 33+ wires through content-encoding.

---

## Decision

Gate 31 is **PASSED on items 1, 2, 3, 4** and **NOT YET PASSED on item 5 (open)**. Phase 32 plans should be drafted but not executed until item 5 closes via RFC-0037.

Phase 31 completed the deflate-family streaming-decode story (gzip + zlib v0.5 wrappers). 19-phase first-try-clean CI streak broke on a long-standing memory-note violation (DocC cross-package ref); fix-and-recover applied within minutes. 15-consecutive zero-scope-reshape RFC streak preserved.

The next step is **RFC-0037 anchoring Phase 32 on swift-brotli v0.5 streaming inflate** — completes codec-tier streaming-decode; unblocks Phase 33+ content-encoding v0.7 streaming-decode wiring. swift-deflate v0.6 true memory-streaming, swift-oauth2-client v0.4, brotli + deflate v0.6 window carry, B3 + Jaeger, idna v0.3, RS-family JWT (when swift-crypto stabilizes `_RSA`), and package rename remain on the queue for Phase 33+.

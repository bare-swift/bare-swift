# Gate 33 Retrospective: Phase 33 → Phase 34

**Date:** 2026-05-18
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 33 (the swift-content-encoding v0.7 streaming-decode wiring anchored by [RFC-0038](../../rfcs/0038-phase-33-anchor-swift-content-encoding-v0.7-streaming-decode.md)) against the Gate 1–32 criteria template and recommends whether Phase 34 should start. Phase 33 was a **single-tranche existing-package minor-bump phase** (33A swift-content-encoding v0.7 — single + multi-coding decode in one combined tranche per Shape B).

**Notable:** Phase 33A closes the HTTP body streaming story end-to-end. Encoder side complete via Phase 28; decoder side complete via Phase 33. Codec-tier + content-encoding tier both stream-capable in both directions.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All anchored Tranche 33A deliverables shipped, CI green, DocC live, index bumped | ✓ PASS |
| 2 | RFC-0038 commitments fulfilled | ✓ PASS *(zero scope reshape; 17th consecutive clean RFC since Phase 17)* |
| 3 | ≥1 substantive RFC accepted *or documented absence of policy gaps* | ✓ PASS (no new RFCs needed; one new sub-pattern codified: inverse-cascade) |
| 4 | RFC-0001 conventions stress-test against Phase 33 deliverables | ✓ DONE BELOW |
| 5 | Phase 34 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0039 below) |

**Overall:** Items 1–4 pass. Item 5 is open. Phase 34 plans should be drafted but not executed until item 5 closes via RFC-0039.

Phase 33 calendar time was ~50-60 min wall-clock (mid-bracket per RFC-0038's 1-2 hour Shape B estimate).

---

## Item 1 — Phase 33 deliverables shipped ✓ PASS

| Tranche | Package | Tag | New tests | DocC | Index | CI |
|---|---|---|---|---|---|---|
| 33A | swift-content-encoding | **v0.7.0** | 25 (91 total) | ✓ | ✓ (0.6.0→0.7.0) | green (first-try) |

Single combined tranche (Shape B per RFC-0038 alternate). **HTTP body streaming story COMPLETE end-to-end.**

**Phase 33 final tally:**
- 33A `ContentEncoding.Streaming.Decoder` struct (Sendable value-type; init/update/finish; reverse-order finish-time cascade).
- Internal `InnerCoding` enum dispatch mirrors v0.6 Encoder pipeline over four codec streaming decoders + identity buffering passthrough.
- New error case `ContentEncodingError.decoderFinished`.
- Codec deps bumped 0.4→0.5 (brotli/gzip/zlib) and 0.2→0.5 (deflate) in lockstep.
- ~155 LOC source + ~365 LOC tests.
- ~50-60 min wall-clock — mid-bracket per RFC-0038.
- **First-try-clean CI streak rebuilding at 2** (Phase 32A + 33A after Phase 31A break).
- **17-consecutive zero-scope-reshape RFC streak** preserved.
- v0.1-v0.6 byte-for-byte preserved; new error case additive.

**Streaming story status (post-Phase 33A) — COMPLETE end-to-end:**
- ✓ Codec-tier streaming-encode: brotli/deflate/gzip/zlib v0.3+
- ✓ Codec-tier `drain()` for cascaded composition: v0.4
- ✓ Codec-tier streaming-decode: brotli/deflate/gzip/zlib v0.5 (buffering wrap)
- ✓ Content-encoding single-coding streaming-encode: v0.5
- ✓ Content-encoding multi-coding streaming-encode: v0.6 (cascaded drain pipeline)
- ✓ **Content-encoding single + multi-coding streaming-decode: v0.7 (reverse-order finish-time cascade)**
- 🟡 v0.6+: codec-tier true memory-streaming inflate (state-machine refactors; demand-driven; would propagate to content-encoding via dep bump)

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists **49 packages** — no count change.

---

## Item 2 — RFC-0038 commitments ✓ PASS (zero scope reshape)

[RFC-0038](../../rfcs/0038-phase-33-anchor-swift-content-encoding-v0.7-streaming-decode.md) committed to:

| RFC-0038 commitment | Outcome |
|---|---|
| One existing package: swift-content-encoding v0.6 → v0.7 | ✓ shipped. |
| `ContentEncoding.Streaming.Decoder` mirroring v0.6 Encoder shape (init/update/finish) | ✓ shipped. |
| InnerDecoder dispatch enum mirroring v0.6 InnerEncoder pattern | ✓ shipped as `InnerCoding` (kept v0.6 naming for symmetry). |
| Shape A (single-coding only, 1 tranche) or Shape B (single + multi-coding, 2 tranches OR 1 combined) — brainstorm decides | ✓ **Shape B as single combined tranche** chosen per reality-check (codec decoders are buffering-wrap, so multi-coding decode collapses to finish-time cascade — no need to split). |
| `ContentEncodingError.decoderFinished` error case | ✓ shipped. |
| Calendar estimate **deferred to brainstorm reality-check** | ✓ honored. Actual ~50-60 min within RFC-0038's 1-2 hour bracket. |
| ~12-15 tests for Shape A; +3-4 for Shape B | ✓ 25 new tests (Shape B; above RFC range — exhaustive coverage of 6 codings + 4 multi-coding chains + edges/errors). |
| Codec deps bumped to 0.5 floors in lockstep | ✓ honored. brotli/gzip/zlib 0.4→0.5; deflate 0.2→0.5. |
| Non-breaking on v0.1-v0.6 | ✓ confirmed. |
| Cross-package DocC discipline | ✓ honored — Decoder docstring uses only in-package symbol refs; codec-namespace refs in Documentation.docc/ContentEncoding.md untouched. |

**Zero scope reshape this phase.** **17th consecutive clean-from-RFC phase** (Phases 17-33).

**Estimate-vs-actual:** RFC-0038 said "~1-2 hours for Shape B (single + multi-coding)"; actual ~50-60 min (lower-mid bracket). 12th successful reality-check application.

---

## Item 3 — Revised criterion ✓ PASS

**Pre-Phase 33 (Gate 32 closeout):** RFC-0038 accepted 2026-05-18.

**During Phase 33 execution:** zero RFCs accepted.

Phase 33 codified one new sub-pattern and reinforced existing patterns:

- **NEW sub-pattern: inverse-cascade.** When a streaming-encode multi-stage pipeline exists (`pipeline[0..N-1]` applied left-to-right, each yielding via drain), the streaming-decode mirror **reverses stage order** (`pipeline = codings.reversed()`) AND **collapses to finish-time-only** when downstream stages don't support per-chunk yielding. `update(_:)` routes raw compressed input to `pipeline[0]` (last-applied coding = decoded first); `finish()` runs each stage's `finish()` and pipes into next stage's `update + finish`. This is the encode-cascade pattern (Phase 28) with role inversion — codified now that we've shipped both encode and decode cascades.
- **Honest-scope-under-limitation at 6 instances** (Phase 25 → 28 → 30 → 31 → 32 → 33). Procedural law fully canonical. Content-encoding v0.7 inherits buffering-wrap from codec v0.5 decoders; documented honestly. Adopters can now expect "v0.5/v0.7 streaming-decode = buffering-wrap until codec v0.6+" without surprise.
- **Brainstorm-empowered-by-RFC scope simplification at 3 instances** (Phase 30 + 32 + 33). Now firmly canonical sub-pattern. Each instance, RFC offered Option A/B; reality-check simplified to one of them (or in Phase 33's case, collapsed Shape B's 2-tranche assumption into 1 combined tranche).
- **Reality-check-before-RFC-estimate at 12/12 successful applications.** Procedural law.
- **Coordinated-version-bump-across-deps at 5 instances** (Phase 24, 27, 28, 31, 33). Procedural law.
- **DocC discipline maintained.** Phase 33A used in-package symbol refs for the new Decoder docstring; cross-package refs in Documentation.docc/ContentEncoding.md (for codec types in topic groups) were left untouched. First-line-memory-note-as-checklist applied.

None warrant a new cross-package RFC. The inverse-cascade sub-pattern is documented in this retrospective and Phase 33A memory closure; future streaming-decode work can reference it without a dedicated RFC.

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

| Convention | swift-content-encoding v0.7 |
|---|---|
| Module name PascalCase | ✓ ContentEncoding |
| Apache-2.0 + LLVM exception SPDX | ✓ |
| Sendable-clean | ✓ |
| Single public error enum | ✓ ContentEncodingError (6 cases: 5 prior + 1 new `decoderFinished`) |
| Public APIs Foundation-free | ✓ |
| README + DocC updated | ✗ (README not updated for v0.7; only CHANGELOG + DocC.docc covered) — minor docs gap |
| CHANGELOG with v0.7.0 entry | ✓ (includes honest-scope note, dep-bump record, migration path) |
| CI green | ✓ first-try |
| DocC bundle | ✓ first-try (`swift package generate-documentation --warnings-as-errors` clean local) |
| Sanitizers ON | ✓ (TSan + ASan both green in CI) |
| Backwards-compat preservation | ✓ v0.1-v0.6 byte-for-byte; new error case additive |

### Deviations and findings

**1. README update not shipped.** swift-content-encoding v0.7 got CHANGELOG + DocC update but no README example for streaming decode. **Tradeoff:** matches the **README-deferral-during-orchestration-sweeps pattern at 4th instance** (Phase 27, 31, 32, 33). Streaming-decode is integrated by HTTP middleware adopters who consume through Streaming.Decoder directly; the existing README's encode/decode examples are sufficient for first-pass discovery. **Memory:** acceptable trade-off; revisit if README discoverability surfaces as a friction point.

**2. Honest-scope-under-limitation pattern at 6 instances.** Procedural law fully canonical. Content-encoding v0.7 documents the buffering-wrap inheritance from codec v0.5 explicitly.

**3. DocC discipline held.** Phase 33A used in-package symbol refs for the new Decoder docstring. The Documentation.docc/ContentEncoding.md topic groups continue to reference codec-tier types — those were not modified in this phase. **Note for Phase 34+:** if memory-streaming codec v0.6+ work lands, Documentation.docc will need cross-package single-backtick refs to discuss the upgrade path. Apply the checklist.

**4. Inverse-cascade sub-pattern codified.** Adds to the set of well-understood multi-stage streaming patterns:
   - Phase 25: `InnerEncoder` single-stage dispatch (single-coding streaming-encode).
   - Phase 28: cascaded encode pipeline via per-chunk drain (multi-coding streaming-encode).
   - Phase 33: reverse-order finish-time cascade (single + multi-coding streaming-decode).
   These three together form the streaming HTTP body story toolkit.

**5. Codec dep-bump-in-lockstep at 5th instance.** Procedural law. All four codec deps bumped simultaneously in a single Package.swift edit.

### Verdict

RFC-0001 held cleanly. One minor docs gap (README deferred — 4th instance of established pattern). Phase 33 closes the HTTP body streaming story end-to-end; future work in this area is **demand-driven optimization** (true memory-streaming refactor at codec tier) rather than capability-blocking.

---

## Item 5 — Phase 34 anchor decision ✗ NOT YET (recommendation)

Phase 34 anchor candidates surveyed:

| Wave | Packages | Risk | Adoption fit | Deferral count |
|---|---|---|---|---|
| **swift-deflate v0.6 true memory-streaming inflate** | deflate v0.6 | High (state-machine refactor of internal Inflater) | **High** — unlocks gzip/zlib v0.6 inheritance + content-encoding true memory-streaming; would propagate the v0.5 honest-scope-under-limitation resolution through the whole stack. | 2 (Phase 32, 33) |
| swift-brotli v0.6 true memory-streaming inflate | brotli v0.6 | High (state-machine refactor; more complex than deflate's Inflater — metablock structure, multiple sub-decoders) | Medium — independent of deflate; could ship in parallel or after. | 1 (Phase 33) |
| swift-oauth2-client v0.4 | oauth2-client v0.4 | Medium | Medium — v0.3 shipped Phase 29; some adopter time elapsed. Active wrapper / JWKS / mTLS / private_key_jwt / device-code candidates. | 2 (Phase 32, 33) |
| swift-brotli v0.6 + swift-deflate v0.6 window carry | 2 packages | Medium | Low-medium — ratio improvement only. | 9 each (Phases 25-33) |
| swift-distributed-tracing-bridge v0.3 (B3 + Jaeger) | bridge v0.3 | Low-medium | Low — 14 deferrals; no concrete demand. | 14 |
| swift-idna v0.3 (Bidi + ContextJ + ContextO) | swift-idna v0.3 | Medium | Low — rarely-hit Unicode rules. | 18 |
| RS-family JWT algorithms | swift-jwt-verify v0.4 | High | Low — **STILL BLOCKED** on swift-crypto `_RSA` SPI. | n/a (blocked) |
| Crypto-adjacent | various | Medium-low | Low — 29x rejected. | 29 (rejected) |
| Package rename `swift-jwt-verify` → `swift-jwt` | rename project | Low | Low-medium | n/a |

### Recommendation: **Anchor on swift-deflate v0.6 (true memory-streaming inflate)**

Reasons:

1. **Resolves the 6-instance honest-scope-under-limitation pattern at its root.** Codec-tier v0.5 buffering-wrap was an honest deferral; v0.6 resolves it via state-machine refactor of the internal Inflater. The streaming-symmetric API surface (init/update/finish) is unchanged — adopters pay zero migration cost.
2. **Unlocks downstream packages.** swift-gzip v0.6 + swift-zlib v0.6 inherit true memory-streaming automatically (they wrap Deflate.Streaming.Decoder per Phase 31 pattern). swift-content-encoding v0.8 inherits via dep bump.
3. **Validates the streaming-symmetric API surface claim.** v0.5 shipped the streaming API surface ahead of true streaming; v0.6 confirms the design holds.
4. **High audience continuity.** Same HTTP-codec adopters from Phases 22-33 streaming sweeps. Adopters get the implementation depth upgrade for free (no API changes).
5. **State-machine-work calibration bucket (3-5 hr from Phase 29).** Brainstorm-empowered-by-RFC could reality-check this estimate; possibly trimmable.
6. **Non-breaking additive.** v0.1-v0.5 APIs unchanged. The buffering-wrap path can be deleted internally; the public surface stays.

### Alternative: anchor on swift-brotli v0.6 instead

Same logic applies but for brotli. Brotli's Decoder.swift is more complex than deflate's Inflater (metablock structure, multiple sub-decoders, supporting helpers like MetaBlockHeader / OutputBuffer / ContextMap / PrefixCode). State-machine refactor is higher risk. **Defer to Phase 35** unless Phase 34 brainstorm uncovers a tractable approach.

### Phase 34 tranche sketch (subject to formal RFC + brainstorm reality-check)

**Shape A (recommended pending reality-check): single tranche, deflate v0.6 state-machine refactor.**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 34A | swift-deflate v0.6 | Refactor internal Inflater to a state-machine yielding bytes incrementally; preserve public Deflate.Streaming.Decoder API shape; delete buffering-wrap path; tests pass | ~3-5 hours (pending reality-check) |

**Shape B (alternate): 2-3 tranche cross-package phase — deflate + gzip + zlib all v0.6 simultaneously.**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 34A | swift-deflate v0.6 | state-machine refactor | ~3-5 hours |
| 34B | swift-gzip v0.6 | dep bump to deflate 0.6; verify wrap inheritance | ~30-60 min |
| 34C | swift-zlib v0.6 | dep bump to deflate 0.6; verify wrap inheritance | ~30-60 min |

Shape B is the cross-package coordinated-version-bump that Phase 31 demonstrated. Decision deferred to brainstorm reality-check.

### Why other waves were rejected for Phase 34

- **swift-brotli v0.6 true memory-streaming inflate** — higher risk than deflate; defer to Phase 35+.
- **swift-oauth2-client v0.4** — v0.3 shipped Phase 29; some settle time but no concrete adopter demand surfaced. Worth Phase 35+ if demand surfaces.
- **brotli / deflate v0.6 window carry** — ratio improvement; not blocking.
- **B3 + Jaeger, idna v0.3** — long-running correct-deferrals; no demand.
- **RS-family JWT** — blocked on swift-crypto `_RSA` SPI.
- **Crypto-adjacent** — 29th rejection.
- **Package rename** — premature.

**Action:** author the anchor decision as **RFC-0039** (Phase 34 anchor: swift-deflate v0.6 true memory-streaming inflate) and accept it before Phase 34 plans start.

---

## Open work to clear Gate 33

1. **Write RFC-0039** (Phase 34 anchor: swift-deflate v0.6 true memory-streaming inflate). ~1-hour task.
2. **Begin Phase 34 brainstorm + reality-check** after RFC-0039 accepts. Brainstorm should:
   - Survey swift-deflate v0.5 internal Inflater state surface (loop variables, output buffer, Huffman code tables).
   - Estimate refactor scope under state-machine pattern. State-machine-work calibration is 3-5 hr from Phase 29.
   - Decide Shape A (deflate only, 1 tranche) vs Shape B (deflate + gzip + zlib coordinated, 2-3 tranches).
   - **Confirm streaming-symmetric API surface is preservable** — this is the validation goal of Phase 34.
   - Maintain DocC discipline: cross-package refs in any updated docstrings use single-backtick.

---

## What this retrospective changes for Phase 34

- The plan-then-execute-inline workflow stays.
- **CI first-try-clean streak at 2** (Phase 32A + 33A; rebuilding after Phase 31A break).
- **Honest-scope-under-limitation pattern at 6 instances** — procedural law fully canonical. Phase 34 is the first opportunity to **resolve** a long-standing instance (codec-tier v0.5 buffering-wrap → v0.6 true memory-streaming).
- **Brainstorm-empowered-by-RFC scope simplification at 3 instances** — canonical sub-pattern.
- **Inverse-cascade sub-pattern codified** — joins single-stage dispatch (Phase 25) and cascaded encode pipeline (Phase 28) in the streaming HTTP body toolkit.
- **Reality-check-before-RFC-estimate at 12/12.**
- **Coordinated-version-bump-across-deps at 5 instances.**
- **README-deferral-during-orchestration-sweeps at 4 instances** — procedural acceptance.
- **HTTP body streaming story COMPLETE end-to-end.** Phase 34+ is optimization (true memory-streaming) and integration (downstream adopters).

---

## Decision

Gate 33 is **PASSED on items 1, 2, 3, 4** and **NOT YET PASSED on item 5 (open)**. Phase 34 plans should be drafted but not executed until item 5 closes via RFC-0039.

Phase 33 closed the HTTP body streaming story end-to-end via content-encoding v0.7 (single + multi-coding streaming-decode in one combined tranche). 17-consecutive zero-scope-reshape RFC streak preserved. First-try-clean CI streak rebuilding at 2.

The next step is **RFC-0039 anchoring Phase 34 on swift-deflate v0.6 true memory-streaming inflate** — resolves the 6-instance honest-scope-under-limitation pattern at the codec-tier foundation; unlocks gzip/zlib v0.6 inheritance + content-encoding v0.8 propagation; validates the streaming-symmetric API surface claim from Phase 30 onward. swift-brotli v0.6 (more complex state-machine refactor), swift-oauth2-client v0.4, brotli + deflate v0.6 window carry, B3 + Jaeger, idna v0.3, RS-family JWT (when swift-crypto stabilizes `_RSA`), and package rename remain on the queue for Phase 35+.

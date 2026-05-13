# Gate 12 Retrospective: Phase 12 → Phase 13

**Date:** 2026-05-13
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 12 (the swift-brotli v0.2 encoder wave anchored by [RFC-0017](../../rfcs/0017-phase-12-anchor-brotli-encoder.md)) against the Gate 1–11 criteria template and recommends whether Phase 13 should start. Phase 12 was a **single-anchor phase** (per RFC-0017) — swift-brotli v0.2 encoder was the headline; 12B was an optional small follow-on. Both shipped.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All anchored Tranche 12A/12B deliverables shipped at v0.2 / v0.4, CI green, DocC live, listed | ✓ PASS |
| 2 | RFC-0017 commitments fulfilled (swift-brotli encoder + swift-content-encoding `br` encode branch) | ✓ PASS |
| 3 | ≥1 substantive RFC accepted *or documented absence of policy gaps* | ✓ PASS (documented absence of policy gaps; see below) |
| 4 | RFC-0001 conventions stress-test against Phase 12 deliverables | ✓ DONE BELOW |
| 5 | Phase 13 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0018 below) |

**Overall:** Items 1, 2, 3, 4 pass cleanly. Item 5 is open. Phase 13 plans should be drafted but not executed until item 5 closes via RFC-0018.

The roadmap's stop conditions did not trigger: Phase 12 calendar time was ~1 day wall-clock (with debugging). No security incidents. No Apple announcements that subsumed planned packages.

---

## Item 1 — Phase 12 deliverables shipped ✓ PASS

| Tranche | Package | Tag | New code | DocC | Index | CI |
|---|---|---|---|---|---|---|
| 12A | swift-brotli | v0.2.0 | ~1700 LOC encoder + 50 new tests | ✓ | ✓ | green (after FORMAT_CL_SPACE fixup + re-tag) |
| 12B | swift-content-encoding | v0.4.0 | ~30 LOC + 4 tests | ✓ | ✓ | green |

Both tranches shipped. **swift-brotli v0.2 closes Phase 10's decoder-first staging**: the encoder lands alongside the v0.1 decoder, completing brotli's bidirectional support. **swift-content-encoding v0.4** adds the `br` branch to its encode multiplexer, closing the symmetric encode/decode story across all five web codings.

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists **43 packages** (unchanged; Phase 12 was version-bumps only).

**Calendar time:** Phase 12 executed in ~1 day wall-clock. swift-brotli v0.2 was the densest encoder effort of any v0.x in the ecosystem (~1700 LOC across 8 new files). swift-content-encoding v0.4 was ~30 LOC plus a dep bump and 1 additive error-enum case.

---

## Item 2 — RFC-0017 commitments ✓ PASS

[RFC-0017](../../rfcs/0017-phase-12-anchor-brotli-encoder.md) committed to a single-anchor Phase 12:

| RFC-0017 commitment | Outcome |
|---|---|
| Tranche 12A: swift-brotli v0.2 encoder (~1500 LOC estimate) | ✓ shipped — actual ~1700 LOC |
| One-shot encoder (no streaming) | ✓ honored |
| Quality 0..11 accepted, scaling match-search depth | ✓ honored |
| Round-trip vectors against v0.1 decoder | ✓ honored — 5 qualities × 7 input shapes |
| Cross-validation against reference `brotli` CLI at ship time | ✓ honored — 4 scenarios pass via `brotli -d` |
| Tranche 12B *(optional)*: swift-content-encoding v0.4 br-encode branch | ✓ shipped — ~30 LOC source + 1 new error case |
| Foundation-free public API; swift-bytes the only dep | ✓ honored |
| Anti-goals: no streaming, no static-dictionary search, no multi-metablock, no NTREESL>1, no block-switch, no last-4-distance shortcuts | ✓ all honored — explicit non-goals listed in CHANGELOG |
| No new packages | ✓ honored — ecosystem stays at 43 |

Notable findings beyond the RFC's stated commitments:

- **LOC estimate was close (1700 vs 1500).** Reasonable accuracy — within 15%.
- **Six discrete encoder bugs surfaced during ship** (documented in [[project_phase12_12a_shipped]] memory). The most notable was **FORMAT_CL_SPACE on uniform-length alphabets**: when all used alphabet symbols share the same Huffman length value (e.g., 256 random literal bytes ≈ uniform freq → all length 8), the meta-Huffman over the 18 length-codes has only 1 used symbol, whose Kraft contribution `32 >> L` can't reach 32. The reference decoder rejects with FORMAT_CL_SPACE. Fix: `ensureLengthVariety` perturbs 3 symbols (1 down to L-1, 2 up to L+1; net Kraft change zero). This was flaky (~70% repro on random-data tests) and surfaced AFTER initial tag; required a same-day re-tag of v0.2.0.
- **Three more subtle wire-format bugs**: simple-form NSYM=3/4 symbol-order mismatch with canonical Huffman lengths; single-symbol tree must emit 0 bits per occurrence; degenerate empty-distance tree fallback. All documented as memories.

---

## Item 3 — Revised criterion (≥1 RFC accepted *or* documented absence of policy gaps) ✓ PASS

**Pre-Phase 12 (Gate 11 closeout):** RFC-0017 (Phase 12 anchor) accepted 2026-05-12.

**During Phase 12 execution:** zero RFCs accepted.

The revised criterion (introduced at Gate 9, confirmed at every gate since) accepts "documented absence of policy gaps" as equivalent. Phase 12 surfaced six encoder-internal bugs during ship; all are memory-saved transferable lessons rather than cross-package policy gaps:

- Uniform-length-alphabet perturbation → memory `project_phase12_12a_shipped`.
- Simple-form NSYM=3/4 ordering → memory `project_phase12_12a_shipped`.
- Single-symbol tree 0-bit-per-occurrence → memory `project_phase12_12a_shipped`.
- Empty distance-tree fallback → memory `project_phase12_12a_shipped`.
- HuffmanBuilder package-merge replaced with bottom-up min-pair Huffman → memory `project_phase12_12a_shipped`.
- Re-tag protocol for broken releases under 1 hour old + zero adopters → memory `project_phase12_12a_shipped`.

None warrant a cross-package RFC. All are codec-specific lessons for whichever package next implements a length-limited Huffman encoder (a small set: brotli encoder + future zopfli-style compression-quality tuning).

**No policy gaps surfaced.** Criterion satisfied.

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

Per-package review:

| Convention | swift-brotli v0.2 | swift-content-encoding v0.4 |
|---|---|---|
| Module name PascalCase, no `Swift` prefix | ✓ Brotli (unchanged) | ✓ ContentEncoding (unchanged) |
| Apache-2.0 + LLVM exception SPDX | ✓ | ✓ |
| NOTICE crediting upstream | ✓ Google brotli MIT (unchanged from v0.1; encoder is clean-room) | ✓ (unchanged from v0.3) |
| Swift 6.0+ tools version | ✓ | ✓ |
| macOS 14 platform floor | ✓ | ✓ |
| Sendable-clean by default | ✓ | ✓ |
| Strict concurrency in CI | ✓ | ✓ |
| Single public error enum, typed throws | ✓ BrotliError (+2 new cases = 12 total) | ✓ ContentEncodingError (+1 new case = 3 total) |
| Public APIs Foundation-free | ✓ | ✓ |
| Repo skeleton from `bare-swift new` | ✓ (existing repo) | ✓ (existing repo) |
| README tagline + ≤30-line example | ✓ updated with v0.2 compress example | ✓ updated with `br` encode mention |
| CHANGELOG with v0.x entry per Keep-a-Changelog | ✓ | ✓ |
| CI green on macOS + Linux | ✓ (with sanitizers off; same as v0.1) | ✓ |
| DocC bundle | ✓ updated with Compress section | ✓ |

### Deviations and findings

**1. First clean-room encoder for a complex compression algorithm.** The brotli decoder (v0.1) was a clean-room implementation of RFC 7932 § 3-8. The encoder (v0.2) is more involved — it builds prefix codes from frequency data, runs LZ77 match-finding, and frames metablocks. The decoder doesn't have direct analogs for these. The convention that emerged:

  - **Always use complex form for ≥ 5 distinct used symbols.** Simple form's NSYM=3 implicit lengths (1,2,2) and NSYM=4 trees (2,2,2,2)/(1,2,3,3) require canonical Huffman lengths to map to SORTED-INDEX positions, which canonical Huffman doesn't guarantee. v0.2 restricts simple form to NSYM ∈ {1, 2} and the trivially-matching NSYM=3/4 cases.
  - **Verify Kraft equality at encode time, not just decode time.** v0.2 has `precondition(checkKraft(...))` guards on every tree it builds. Catches HuffmanBuilder regressions immediately rather than producing malformed output the decoder rejects with `.invalidPrefixCode`.
  - **Ensure ≥ 2 distinct length values in the alphabet** to avoid meta-Huffman degeneration on uniform distributions (random binary inputs).

  Future codec encoders (deflate v0.3, gzip v0.3, zlib v0.3 — if encoder-quality tuning is ever added) should reuse this pattern.

**2. Re-tag protocol for short-lived broken releases.** v0.2.0's initial tag had the FORMAT_CL_SPACE bug. Cleanup process (now canonical for <1-hour-old releases with zero adopters):

  - `gh release delete vN.N.N --cleanup-tag`
  - Push fixed commit on `main`
  - `git tag -a` + `git push origin vN.N.N` with same tag name
  - Wait for CI + verify docs HTTP 200

  For releases > 1 hour old or with any known adopter, cut a patch (`vN.N.N+1`) instead. Memory documents both protocols.

**3. Additive error-enum cases as a v0.4 minor.** `ContentEncodingError` gained `.encodingFailed(String)` in v0.4. Since the error enum is non-frozen, this is a SOFT semver bump (exhaustive `switch` consumers must add the new case; existing code that just throws/propagates is fine). v0.4 = minor; this is correct per Keep-a-Changelog + semver.

**4. Patch-release dep-bump pattern continues working.** swift-content-encoding v0.4 follows the same shape as v0.3 (which added `br` decode): bump a downstream dep, add one switch case in the multiplexer, add 1-2 tests, update CHANGELOG. ~30 LOC including the new error case.

### Verdict

RFC-0001 held cleanly under Phase 12 stress. The "single-anchor + small follow-on" phase shape continues to work well — second consecutive phase using this pattern (Phase 10 brotli decoder + content-encoding v0.3, Phase 12 brotli encoder + content-encoding v0.4).

---

## Item 5 — Phase 13 anchor decision ✗ NOT YET (recommendation)

Phase 13 anchor candidates surveyed (drawn from Gate 11 + Gate 12 standing queue):

| Wave | Packages | Risk | Adoption fit |
|---|---|---|---|
| **Internationalization** | swift-publicsuffix, swift-idna | High (UTS #46 + #15 + Punycode; PSL ~150 KiB data) | **High** — closes swift-uri's IDNA deferral; unblocks correct hostname comparison in HTTPS clients. Named as strongest Phase 13 signal by both Gate 11 + Gate 12. |
| OTLP cross-signal v0.2 | swift-otlp-traces v0.2, swift-otlp-logs v0.2, swift-otlp-json | Medium | Medium — narrower audience than i18n. |
| swift-jwt-verify v0.2 (signing + RS256) | swift-jwt-verify v0.2 | Medium | Medium-low — fresh from v0.1; let it settle. |
| OAuth 2.0 client primitives | swift-oauth2-client | Medium | Low-medium — no concrete adopter demand yet. |
| Crypto-adjacent | swift-blake3, swift-siphash, swift-hkdf | Medium-low | Low — tenth rejection candidate. |

### Recommendation: **Anchor on internationalization (PSL + IDNA)**

Reasons:

1. **Two-gate-deep consensus.** Gate 11 retro flagged internationalization as the strongest Phase 13 candidate; Gate 12 retro (this doc) confirms. Three consecutive gates that have considered i18n haven't found a reason to defer further.
2. **Closes the longest-standing deferral with concrete impact.** swift-uri shipped with explicit IDNA deferral in Phase 4. Hostname comparison in HTTPS clients (TLS certificate matching, CORS origin validation, cookie domain matching) all require IDNA-correct comparison. Without it, swift-uri users can't safely compare hostnames across Unicode normalization forms.
3. **Audience continuity.** Same HTTP server/client implementers Phases 4 / 7 / 8 / 9 / 10 / 11 / 12 serve. PSL + IDNA round out the URI story.
4. **Decomposes naturally into 2 packages.** swift-publicsuffix is data-heavy (~150 KiB PSL with periodic updates) but algorithmically simple (longest-match suffix lookup). swift-idna is data-lighter but algorithmically heavy (UTS #46 + UTS #15 + Punycode). Independent shipping order: 13A swift-publicsuffix (easier; data-driven), 13B swift-idna (harder; spec-heavy), optional 13C swift-uri v0.2 integration.
5. **Risk profile is well-bounded.**
   - swift-publicsuffix: ~400 LOC + 150 KiB data table. Data update is occasional (1-2x/year). LICENSE attribution per RFC-0001's "MIT-licensed data + clean-room code" template (already used for swift-brotli's dictionary).
   - swift-idna: ~800 LOC for UTS #46 + Punycode (RFC 3492). Punycode is ~150 LOC algorithmic. UTS #46 is data-driven (NFC normalization tables + IDNA mapping table). Reference: ICU C source as oracle for test vectors.
   - swift-uri v0.2 (if 13C ships): ~50 LOC adapter wiring swift-idna into hostname comparison.
6. **Sets up Phase 14+ options.** Once i18n ships:
   - **OTLP cross-signal v0.2** becomes the natural Phase 14 anchor.
   - **swift-jwt-verify v0.2 signing** stays available.
   - **OAuth 2.0** stays available.
   - **swift-brotli v0.3 streaming** stays available.

### Phase 13 tranche sketch (subject to formal RFC)

| Tranche | Packages | Estimated calendar |
|---|---|---|
| 13A | swift-publicsuffix v0.1 (PSL longest-match lookup; ~400 LOC + 150 KiB data) | ~1 day |
| 13B | swift-idna v0.1 (UTS #46 + Punycode RFC 3492; ~800 LOC) | ~2-3 days |
| 13C *(stretch)* | swift-uri v0.2 — wire swift-idna into hostname normalization | ~0.5 day |

Total Phase 13 budget: ~3-4 days wall-clock. Resembles Phase 7 (compression decoder wave) and Phase 11 (auth primitives) in shape: 2-3 medium packages + an optional integration follow-on.

### Why other waves were rejected

- **OTLP cross-signal v0.2** — still a fine Phase 14 candidate. Narrower audience than i18n; deferral isn't aging fast.
- **swift-jwt-verify v0.2 signing** — fresh v0.1 from Phase 11. Let it settle and gather adopter feedback before expanding scope.
- **OAuth 2.0 client primitives** — no concrete adopter demand. Premature.
- **Crypto-adjacent** — tenth consecutive rejection across gates.

**Action:** author the anchor decision as **RFC-0018** (Phase 13 anchor: internationalization wave) and accept it before Phase 13 plans start.

---

## Open work to clear Gate 12

1. **Write RFC-0018** (Phase 13 anchor: internationalization). 1-2 hour task. References this retrospective + Gate 11's standing recommendation.
2. **Begin Phase 13 planning** after RFC-0018 accepts.

---

## What this retrospective changes for Phase 13

- The plan-then-execute-with-checkpoints workflow stays.
- **New:** Re-tag protocol formalized for sub-hour broken releases. Memory entry covers the procedure.
- **New:** Encoder-internal Kraft assertions are now standard for any clean-room encoder that builds canonical Huffman codes. Add to bare-swift encoder template (informal).
- **New:** "Ensure ≥ 2 distinct length values in alphabets" perturbation pattern documented — applies to any encoder using complex-form prefix codes (RFC 7932 § 3.5 family, RFC 1951 § 3.2 family).
- **New:** Data-heavy package template established by swift-brotli (122 KiB dictionary) is ready to reuse for swift-publicsuffix (150 KiB PSL). Same `.gitignore` + `scripts/` generation + sanitizer-off pattern.
- **Open question for Phase 13 RFC-0018:** swift-publicsuffix data update cadence. PSL updates ~1-2x/year. Should we set up an automated weekly check + PR, or rely on manual updates? Decision deferred to the RFC.

---

## Decision

Gate 12 is **PASSED on items 1, 2, 3, 4** and **NOT YET PASSED on item 5 (open)**. Phase 13 plans should be drafted but not executed until item 5 closes via RFC-0018.

Phase 12 closed the compression tier's last asymmetry by adding the brotli encoder. After Phase 12, all four web compression codings (identity / gzip / deflate / br) support symmetric encode + decode end-to-end through swift-content-encoding's multiplexer. The longest-standing v0.x deferral in the codec family is now closed.

The next step is **RFC-0018 anchoring Phase 13 on the internationalization wave**, which both clears Gate 12 and closes the longest-standing deferral in the URI family (swift-uri's IDNA gap from Phase 4). OTLP cross-signal, swift-jwt-verify v0.2 signing, OAuth 2.0, and swift-brotli v0.3 streaming remain on the queue for Phase 14+.

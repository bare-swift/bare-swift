# Gate 13 Retrospective: Phase 13 → Phase 14

**Date:** 2026-05-13
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 13 (the internationalization wave anchored by [RFC-0018](../../rfcs/0018-phase-13-anchor-internationalization.md)) against the Gate 1–12 criteria template and recommends whether Phase 14 should start. Phase 13 was a **three-package wave** (13A swift-publicsuffix, 13B swift-idna, 13C swift-uri v0.2 IDNA accessors). All three shipped.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All anchored Tranche 13A/13B/13C deliverables shipped, CI green, DocC live, listed | ✓ PASS |
| 2 | RFC-0018 commitments fulfilled (PSL + IDNA + swift-uri integration) | ✓ PASS *(with documented scope-cut on 13B)* |
| 3 | ≥1 substantive RFC accepted *or documented absence of policy gaps* | ✓ PASS (documented absence of policy gaps) |
| 4 | RFC-0001 conventions stress-test against Phase 13 deliverables | ✓ DONE BELOW |
| 5 | Phase 14 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0019 below) |

**Overall:** Items 1–4 pass. Item 5 is open. Phase 14 plans should be drafted but not executed until item 5 closes via RFC-0019.

The roadmap's stop conditions did not trigger: Phase 13 calendar time was ~1 day for all three tranches (matching RFC-0018's budget). No security incidents. No Apple announcements that subsumed planned packages.

---

## Item 1 — Phase 13 deliverables shipped ✓ PASS

| Tranche | Package | Tag | New code | DocC | Index | CI |
|---|---|---|---|---|---|---|
| 13A | swift-publicsuffix | v0.1.0 | ~400 LOC + 140 KiB PSL data | ✓ | ✓ | green (clean ship) |
| 13B | swift-idna | v0.1.0 | ~400 LOC | ✓ | ✓ | green (after ubuntu-22.04 6.0 infra flake — canceled + re-ran) |
| 13C | swift-uri | v0.2.0 | ~40 LOC + 13 tests | ✓ | ✓ | green (clean) |

All three tranches shipped. The internationalization stack is now functionally complete at v0.1 scope:
- **swift-publicsuffix** answers "what's the public suffix / registrable domain of this hostname?"
- **swift-idna** answers "convert this hostname between Unicode and ACE Punycode forms"
- **swift-uri v0.2** wires both via `URI.Host.asciiDomain` / `URI.Host.unicodeDomain` accessors

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists **45 packages** — 2 new from Phase 13 plus 1 version bump.

**Calendar time:** Phase 13 executed in ~1 day wall-clock. 13A was ~3 hours (data ingestion + algorithm + tests). 13B was ~4 hours (RFC 3492 transliteration; the bias-adaptation arithmetic was the densest part). 13C was ~1 hour (additive shim).

---

## Item 2 — RFC-0018 commitments ✓ PASS (with documented scope-cut)

[RFC-0018](../../rfcs/0018-phase-13-anchor-internationalization.md) committed to a three-tranche internationalization wave:

| RFC-0018 commitment | Outcome |
|---|---|
| Tranche 13A: swift-publicsuffix (PSL longest-match; ~400 LOC + 150 KiB data) | ✓ shipped — actual 140 KiB after comment strip |
| PSL wildcard + exception rules per § 3 | ✓ honored — exceptions trump wildcards per algorithm |
| PSL data update via manual script (1-2x/year cadence) | ✓ honored — `scripts/regenerate-psl.sh` |
| MPL 2.0 data attribution in NOTICE | ✓ honored |
| Tranche 13B: swift-idna (UTS #46 + Punycode RFC 3492; ~800 LOC) | ✗ **scope-cut to Punycode-only** (see below) |
| Tranche 13C *(stretch)*: swift-uri v0.2 wiring | ✓ shipped — `asciiDomain` + `unicodeDomain` accessors |
| Foundation-free public APIs across the wave | ✓ honored |
| No new umbrella tier | ✓ honored — all three packages in existing tiers (`format`) |
| 2 new packages → 45 total | ✓ honored |

**Notable scope-cut on 13B (documented in CHANGELOG + memory):**

RFC-0018 sketched swift-idna at ~800 LOC for "UTS #46 + Punycode + RFC 3492." During brainstorming, reality check showed:
- Reference rust-idna is ~2200 LOC and delegates NFC normalization to a separate `icu_normalizer_data` crate (~150 KiB Unicode tables).
- A real UTS #46 implementation needs the IdnaMappingTable (~50 KiB packed) + NFC Canonical Combining Class + Canonical Decomposition tables (~150 KiB packed).
- Total honest scope for full UTS #46: ~3-5k LOC + ~200 KiB Unicode data.

**Decision:** ship the bedrock Punycode codec at honest scope (~400 LOC, no Unicode tables) as v0.1. Defer UTS #46 mapping + NFC normalization + Bidi/ContextJ to swift-idna v0.2. v0.1 callers must pre-normalize (lowercase + NFC) before calling `toASCII`.

This is a legitimate scope cut — the same shape as swift-brotli v0.2 encoder in Phase 12 (which deferred static-dictionary search, multi-metablock, and other UTS #46-equivalent features to v0.3). RFC-0018's estimate was optimistic; the v0.1 shipped at honest LOC.

**Downstream consequence on 13C:** swift-uri v0.1's parser still throws `URIError.idnaNotSupported` for Unicode hostnames. v0.2's IDNA accessors work on `URI.Host` values (constructed directly or parsed from already-ACE inputs); auto-encode during parse is deferred to swift-uri v0.3 (which needs swift-idna v0.2's full UTS #46 anyway). Documented in CHANGELOG + memory.

---

## Item 3 — Revised criterion (≥1 RFC accepted *or* documented absence of policy gaps) ✓ PASS

**Pre-Phase 13 (Gate 12 closeout):** RFC-0018 (Phase 13 anchor) accepted 2026-05-12.

**During Phase 13 execution:** zero RFCs accepted.

The revised criterion accepts "documented absence of policy gaps" as equivalent. Phase 13 surfaced these findings during execution; all are memory-saved transferable lessons rather than cross-package policy gaps:

- **DocC strict-mode bare-symbol-reference ambiguity** — bare ``toASCII(_:)`` reference in `IDNA`'s Overview prose tries to resolve at current scope and fails. Fix: fully-qualified ``IDNA/toASCII(_:)``. Documented in [[feedback_docc_disambiguation_suffixes]] (existing entry; reinforced).
- **GitHub Actions ubuntu-22.04 Swift 6.0 "Install Swift" infra flake** — hung 51+ minutes on swift-idna's initial CI run. Other 6 jobs succeeded. Canceled + re-ran completed in 2 min. Pure infra issue; no code change. Memory note: don't panic-revert when one Linux runner stalls; cancel + retry.
- **Scope-cut protocol on v0.1 deliverables** — RFC-0018 had an optimistic LOC estimate for swift-idna; brainstorm surfaced the gap; v0.1 shipped at honest scope. Pattern: brainstorm phase MUST sanity-check the LOC estimate by examining a reference implementation before locking spec. Reinforces [[feedback_decide_design_decisions]] but adds the "verify LOC estimate" step.

None of these warrant a new cross-package RFC. **Documented absence of policy gaps. Criterion satisfied.**

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

Per-package review:

| Convention | swift-publicsuffix | swift-idna | swift-uri v0.2 |
|---|---|---|---|
| Module name PascalCase, no `Swift` prefix | ✓ PublicSuffix | ✓ IDNA | ✓ URI (unchanged) |
| Apache-2.0 + LLVM exception SPDX | ✓ | ✓ | ✓ |
| NOTICE crediting upstream | ✓ Mozilla PSL MPL 2.0 (data) | ✓ Apache 2.0 only (no third-party data) | ✓ (unchanged) |
| Swift 6.0+ tools version | ✓ | ✓ | ✓ |
| macOS 14 platform floor | ✓ | ✓ | ✓ |
| Sendable-clean by default | ✓ | ✓ | ✓ |
| Strict concurrency in CI | ✓ | ✓ | ✓ |
| Single public error enum, typed throws | (no errors; non-match is nil) | ✓ IDNAError (4 cases) | ✓ URIError (unchanged) |
| Public APIs Foundation-free | ✓ | ✓ | ✓ |
| Repo skeleton from `bare-swift new` | ✓ | ✓ | ✓ (existing repo) |
| README tagline + ≤30-line example | ✓ | ✓ | ✓ updated with IDNA section |
| CHANGELOG with v0.x entry per Keep-a-Changelog | ✓ | ✓ | ✓ |
| CI green on macOS + Linux | ✓ | ✓ (after infra rerun) | ✓ |
| DocC bundle | ✓ | ✓ | ✓ |
| docc-target set from initial commit (Gate 11 lesson) | ✓ | ✓ | ✓ (unchanged) |
| Sanitizers off if >100 KiB static data (Gate 10 lesson) | ✓ (140 KiB PSL) | ✓ (no large data) | ✓ (unchanged) |

### Deviations and findings

**1. MPL-data + Apache-code attribution template reused.** swift-publicsuffix is the third bare-swift package (after swift-brotli's Google data and swift-jwt-verify's swift-crypto wrapping) to embed third-party-licensed data in a NOTICE-separated form. The template now covers three distinct license shapes:
- Code-derivative MIT (swift-brotli's dictionary)
- Code-wrapping Apache (swift-jwt-verify's swift-crypto)
- Data-only MPL 2.0 (swift-publicsuffix's PSL)

All three follow the same NOTICE pattern: "Code is Apache 2.0 / LLVM exception. Data is <THIRD-PARTY LICENSE>. Clear sentence separating the two." Future packages embedding third-party-licensed content can pattern-match.

**2. Honest-scope v0.1 shipping continues to work cleanly.** swift-idna's Punycode-only v0.1 is the third package this gate cycle to ship at honest scope rather than the RFC's optimistic estimate (swift-brotli v0.2 encoder, swift-content-encoding v0.4 br-encode, swift-idna v0.1 Punycode-only). Pattern: brainstorm phase reality-checks the LOC, ships smaller v0.1, defers ambitious features to v0.2+. Adopters get usable bedrock immediately rather than waiting for "complete."

**3. Additive minor-bump for downstream integration.** swift-uri v0.2.0 added swift-idna as a dep + 2 accessors. All v0.1 tests still pass. This is the third version-bump in 2 gates using the additive-only shape (swift-content-encoding v0.3 br-decode, v0.4 br-encode, now swift-uri v0.2 IDNA accessors). The shape is canonical: bump dep, add focused feature on top, document explicitly what's unchanged.

**4. Parametrized test vectors continue to scale.** swift-idna's `arguments:` parametrization over RFC § 7.1's 11 worked examples × encode+decode = 22 cases caught all bugs on first run. Pattern proven across Phase 11 (jwt-verify) + Phase 13 (publicsuffix Mozilla vectors, idna RFC vectors). Use parametrized arguments for any canonical test-vector suite.

### Verdict

RFC-0001 held cleanly under Phase 13 stress. Three tranches of varying shape (data-heavy package, algorithm-heavy package, additive integration shim) all complied without exception.

---

## Item 5 — Phase 14 anchor decision ✗ NOT YET (recommendation)

Phase 14 anchor candidates surveyed (drawn from Gate 11+12 standing queue plus Phase 13's new deferrals):

| Wave | Packages | Risk | Adoption fit |
|---|---|---|---|
| **OTLP cross-signal v0.2** | swift-otlp-traces v0.2, swift-otlp-logs v0.2, swift-otlp-json | Medium | **High** — strongest queue candidate per Gates 11+12. Closes Phase 2/3/4 single-signal deferral. |
| swift-idna v0.2 (UTS #46 + NFC) | swift-idna v0.2 | High (large Unicode tables + Bidi rule + ContextJ/O) | Medium — closes Phase 13's deferral; needed before swift-uri v0.3 auto-encode. |
| swift-jwt-verify v0.2 (signing + RS256) | swift-jwt-verify v0.2 | Medium | Medium-low — fresh v0.1; let it settle. |
| swift-brotli v0.3 (streaming encoder) | swift-brotli v0.3 | Medium | Medium — narrower than OTLP. |
| OAuth 2.0 client primitives | swift-oauth2-client | Medium | Low-medium — no concrete adopter demand. |
| Crypto-adjacent | swift-blake3, swift-siphash, swift-hkdf | Medium-low | Low — 10th rejection candidate. |

### Recommendation: **Anchor on OTLP cross-signal v0.2**

Reasons:

1. **Two-gate-deep consensus, plus the standing winner.** Gate 11 + Gate 12 both named OTLP as the next-strongest signal after i18n. With i18n now done (Phase 13), OTLP is the leading candidate.
2. **Closes the longest-standing v0.x deferral in the observability tier.** swift-otlp-traces shipped v0.1 in Phase 3 (2026-05-09); swift-otlp-logs shipped v0.1 in Phase 4 (2026-05-10). Both were single-signal. Adding cross-signal (correlating traces ↔ logs ↔ metrics) closes a known limitation.
3. **Audience continuity is good.** Same observability folks who use swift-prometheus / swift-prometheus-metrics / swift-distributed-tracing-bridge. The HTTP-server stack closed Phase 11; observability is the other major adopter community.
4. **Decomposes naturally into 3 packages.**
   - swift-otlp-traces v0.2 — add log + metric correlation (TraceContext IDs in log records, exemplars in metric points).
   - swift-otlp-logs v0.2 — same correlation on the log side.
   - swift-otlp-json (new) — JSON export adapter (alongside the existing protobuf-only exporters). Useful for adopters that need OTLP/HTTP/JSON over HTTP/protobuf.
5. **Risk profile is well-bounded.**
   - swift-otlp-traces / -logs v0.2: ~200 LOC each (TraceContext wiring + exemplar attachment). swift-otlp specs are stable.
   - swift-otlp-json: ~600 LOC (JSON serialization of the same OTLP message types; swift-json is in-house).
6. **Sets up Phase 15+ options.** Once OTLP cross-signal ships:
   - **swift-idna v0.2 (UTS #46)** becomes the natural Phase 15 anchor — completes Phase 13's deferral.
   - **swift-jwt-verify v0.2 signing** stays available.
   - **swift-brotli v0.3 streaming** stays available.
   - **OAuth 2.0** stays available.

### Phase 14 tranche sketch (subject to formal RFC)

| Tranche | Packages | Estimated calendar |
|---|---|---|
| 14A | swift-otlp-json v0.1 (new package; OTLP/HTTP/JSON export) | ~1.5 days |
| 14B | swift-otlp-traces v0.2 (cross-signal correlation: log link IDs) | ~0.5 day |
| 14C | swift-otlp-logs v0.2 (cross-signal correlation: trace link IDs) | ~0.5 day |

Total Phase 14 budget: ~2.5 days wall-clock. Resembles Phase 11 (auth primitives) in shape — multiple small-to-medium tranches with overlapping audience.

Alternate framing: 14A could be a single-anchor (matching Phase 10 / Phase 12 shape) with 14B/14C as optional follow-ons.

### Why other waves were rejected

- **swift-idna v0.2 (UTS #46)** — large Unicode tables + spec complexity; better as Phase 15 single-anchor once OTLP cross-signal closes. Phase 13's deferral isn't aging fast (no adopter complaints yet).
- **swift-jwt-verify v0.2 signing** — fresh v0.1 (Phase 11); let it settle.
- **swift-brotli v0.3 streaming** — narrower than OTLP; deferral isn't aging fast.
- **OAuth 2.0 client primitives** — no concrete adopter demand; premature.
- **Crypto-adjacent** — 10th consecutive rejection; swift-crypto stays entrenched.

**Action:** author the anchor decision as **RFC-0019** (Phase 14 anchor: OTLP cross-signal v0.2) and accept it before Phase 14 plans start.

---

## Open work to clear Gate 13

1. **Write RFC-0019** (Phase 14 anchor: OTLP cross-signal). 1-2 hour task. References this retrospective.
2. **Begin Phase 14 planning** after RFC-0019 accepts.

---

## What this retrospective changes for Phase 14

- The plan-then-execute-with-checkpoints workflow stays.
- **Reinforced (not new):** Brainstorm phase MUST verify RFC LOC estimate against a reference implementation before locking spec. Phase 13's swift-idna scope-cut was the third in two gates; pattern is now canonical.
- **Reinforced (not new):** DocC strict-mode bare-symbol references must use either fully-qualified ``Module/symbol(_:)`` form or single-backtick `` `symbol(_:)` `` for non-link inline. Memory entry exists; reinforced by Phase 13.
- **New:** OTLP encoders v0.2 will be the first packages to gain cross-signal correlation. The TraceContext propagation pattern (W3C trace-context header → SpanContext IDs → exemplar attachments on metric points and log records) is well-specified; v0.2 just wires the public APIs.
- **Carried forward:** Phase 13 left three documented deferrals (swift-idna v0.2 UTS #46, swift-uri v0.3 auto-encode, swift-publicsuffix v0.2 ICANN/PRIVATE split). None block Phase 14; all are queued for Phase 15+.

---

## Decision

Gate 13 is **PASSED on items 1, 2, 3, 4** and **NOT YET PASSED on item 5 (open)**. Phase 14 plans should be drafted but not executed until item 5 closes via RFC-0019.

Phase 13 closed the internationalization wave at v0.1 scope — swift-uri's Phase-4-era IDNA deferral is partially closed (read side complete; write side awaits swift-idna v0.2). The bare-swift URI / hostname story now has three composable pieces: PSL lookup, IDNA codec, URI accessors.

The next step is **RFC-0019 anchoring Phase 14 on OTLP cross-signal v0.2 (with optional swift-otlp-json v0.1)**, which both clears Gate 13 and starts deepening the observability tier. swift-idna v0.2 UTS #46, swift-jwt-verify v0.2 signing, swift-brotli v0.3 streaming, and OAuth 2.0 remain on the queue for Phase 15+.

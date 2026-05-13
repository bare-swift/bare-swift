# Gate 14 Retrospective: Phase 14 → Phase 15

**Date:** 2026-05-13
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 14 (the OTLP cross-signal wave anchored by [RFC-0019](../../rfcs/0019-phase-14-anchor-otlp-cross-signal.md)) against the Gate 1–13 criteria template and recommends whether Phase 15 should start. Phase 14 was a **three-tranche wave** (14A swift-otlp-json, 14B swift-tracing-otlp v0.3, 14C swift-log-otlp v0.3). All three shipped.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All anchored Tranche 14A/14B/14C deliverables shipped, CI green, DocC live, listed | ✓ PASS |
| 2 | RFC-0019 commitments fulfilled (OTLP/HTTP/JSON + TraceContext + cross-signal log correlation) | ✓ PASS *(with documented RFC doc-errors on names + versions)* |
| 3 | ≥1 substantive RFC accepted *or documented absence of policy gaps* | ✓ PASS (documented absence of policy gaps) |
| 4 | RFC-0001 conventions stress-test against Phase 14 deliverables | ✓ DONE BELOW |
| 5 | Phase 15 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0020 below) |

**Overall:** Items 1–4 pass. Item 5 is open. Phase 15 plans should be drafted but not executed until item 5 closes via RFC-0020.

The roadmap's stop conditions did not trigger: Phase 14 calendar time was ~1 day for all three tranches (matching RFC-0019's 2.5-day budget, actually under). No security incidents. No Apple announcements that subsumed planned packages.

---

## Item 1 — Phase 14 deliverables shipped ✓ PASS

| Tranche | Package | Tag | New code | DocC | Index | CI |
|---|---|---|---|---|---|---|
| 14A | swift-otlp-json | v0.1.0 | ~770 LOC + 40 tests | ✓ | ✓ | green (clean first-push) |
| 14B | swift-tracing-otlp | **v0.3.0** | ~130 LOC + 13 tests | ✓ | ✓ | green (clean first-push) |
| 14C | swift-log-otlp | **v0.3.0** | ~50 LOC + 6 tests | ✓ | ✓ | green (clean first-push) |

All three tranches shipped. The OTLP observability stack is now feature-complete at v0.3 scope:
- **swift-otlp-json** emits OTLP/HTTP/JSON for all three signals (traces, logs, metrics), reusing the existing `OTLP.*` Swift message types.
- **swift-tracing-otlp v0.3** exposes `OTLP.TraceContext` — a W3C Trace Context value with strict `traceparent` serialize/parse.
- **swift-log-otlp v0.3** offers an `OTLP.LogRecord(..., traceContext:)` convenience init that fills the correlation fields (`traceID` / `spanID` / `flags`) from a single value.

End-to-end cross-signal correlation now works: parse inbound `traceparent` → `OTLP.TraceContext` → use it on both `OTLP.Span` and `OTLP.LogRecord` → emit via JSON or protobuf.

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists **46 packages** — 1 new from Phase 14 plus 2 version bumps.

**Calendar time:** Phase 14 executed in ~1 day wall-clock (well under RFC-0019's 2.5-day estimate). 14A was the largest at ~3 hours (7 source files, hand-rolled JSON writer, 40 tests). 14B was ~1 hour (single value type + parser + 13 tests). 14C was ~30 min (single convenience init + 6 tests). The plan-then-execute-inline workflow on 14C used the `superpowers:writing-plans` + `superpowers:executing-plans` skills end-to-end.

**Cleanest CI shipping cycle across the project so far** — all three tranches green on first push for both main commits and tag-triggered Release / Publish-docs workflows. Zero fixup commits.

---

## Item 2 — RFC-0019 commitments ✓ PASS (with documented doc-errors)

[RFC-0019](../../rfcs/0019-phase-14-anchor-otlp-cross-signal.md) committed to a three-tranche OTLP cross-signal wave:

| RFC-0019 commitment | Outcome |
|---|---|
| Tranche 14A: swift-otlp-json (OTLP/HTTP/JSON encoder; ~600 LOC) | ✓ shipped — actual ~770 LOC (proto3 JSON mapping rules: 64-bit-as-string, NaN/Inf-as-string, bytes-as-base64). |
| `OTLPJSON.encode(_:)` for traces, logs, metrics | ✓ shipped as `OTLPJSON.encodeTraces / encodeLogs / encodeMetrics`. |
| `OTLPJSONError` typed-throws enum | ✗ **omitted** (encoding pure value types has no failure modes; non-throwing API). |
| Reuse OTLP message types from swift-otlp-traces / -logs / -metrics | ✓ honored. **Note: RFC named these wrong** — actual modules are `TracingOTLP` / `LogOTLP` / `OTLPExporter`. |
| Tranche 14B: swift-tracing-otlp **v0.2** TraceContext propagation hooks | ✓ shipped — **as v0.3** (package was already at v0.2 from Time.Instant integration). |
| `TraceContext.current` accessor reading swift-distributed-tracing context | ✗ **reshaped** — shipped the value type + `traceparent` serialize/parse only. `.current` requires swift-distributed-tracing-bridge wiring which is a separate concern (already partly addressed by swift-distributed-tracing-bridge v0.1 from Phase 3B). |
| W3C trace-context header builders: `TraceContext.traceparent(_:)` / `.tracestate(_:)` | ✓ shipped as `OTLP.TraceContext.traceparent: String?` (computed) + `OTLP.TraceContext.parse(traceparent:)`. tracestate is a field (round-trips), not a builder (W3C tracestate has vendor-specific grammar; out of scope for v0.3). |
| Tranche 14C: swift-otlp-logs **v0.2** TraceContext attachment | ✓ shipped — **as v0.3** (package was already at v0.2 from Time.Instant integration). |
| Automatic reading of `TraceContext.current` when emitting log records | ✗ **reshaped** — shipped as an explicit `OTLP.LogRecord(traceContext:)` convenience init (caller passes the context value). Automatic context-reading bridge would require swift-log frontend integration, which is downstream from the encoder package. |
| All existing v0.1 export APIs unchanged | ✓ honored. All 26 swift-log-otlp v0.2 tests + all 62 swift-tracing-otlp v0.2 tests continue to pass. |
| Foundation-free public APIs across the wave | ✓ honored. |
| 1 new umbrella package | ✓ honored — swift-otlp-json. |
| 1 new package + 2 v0.2 bumps → 46 total | ✓ honored. |

**Notable doc-errors in RFC-0019 (each tranche surfaced one):**

- **14A doc-error: wrong package names.** RFC-0019 called the existing OTLP packages "swift-otlp-traces" and "swift-otlp-logs." The actual names are `swift-tracing-otlp` (module `TracingOTLP`) and `swift-log-otlp` (module `LogOTLP`); shared types live in `swift-otlp-exporter` (module `OTLPExporter`). The error was caught during 14A implementation when type lookups failed; 14A fixed up its imports correctly without code churn.
- **14B/14C doc-error: wrong version bumps.** RFC-0019 called for "v0.2 bumps" but both packages were already at v0.2 from Phase 4's Time.Instant integration (2026-05-10). Actual ships were v0.3.0. The error didn't affect implementation but did affect the umbrella `packages/index.json` entries and the CHANGELOG version headers. Five-minute fix at ship time per tranche.
- **14B/14C scope reshape.** Both shipped as explicit, caller-provides-the-value APIs rather than implicit "read the current swift-distributed-tracing context." This is the **right shape** for an encoder package (decoupled from any specific logging/tracing frontend) but differs from the RFC's sketch. A downstream `swift-distributed-tracing-otlp` adapter (Phase 16+ candidate?) would be the right place for automatic context wiring.

**Pattern for future RFCs (carried forward):**

1. **Verify package names against `packages/index.json` before drafting an RFC** — the umbrella index is the source of truth. A 30-second `grep` would have caught the 14A name error.
2. **Verify current versions before specifying v0.x bumps** — same grep.
3. **Prefer explicit-input API shapes over implicit-context shapes in low-level encoder packages.** Implicit context belongs in adapter packages, not encoders.

None of these warrant a new cross-package RFC. They are reinforcing pattern notes already partly captured in feedback memory.

---

## Item 3 — Revised criterion (≥1 RFC accepted *or* documented absence of policy gaps) ✓ PASS

**Pre-Phase 14 (Gate 13 closeout):** RFC-0019 (Phase 14 anchor) accepted 2026-05-13.

**During Phase 14 execution:** zero RFCs accepted.

The revised criterion accepts "documented absence of policy gaps" as equivalent. Phase 14 surfaced these findings during execution; all are memory-saved transferable lessons rather than cross-package policy gaps:

- **RFC name-and-version drift on existing packages.** As described in Item 2 above. The fix is procedural (verify-before-draft); no new RFC needed.
- **DocC cross-package symbol-link strictness reconfirmed.** swift-tracing-otlp v0.3's DocC catalog initially attempted `` ``OTLPExporter/OTLP/TraceContext`` `` for a type defined in the local module via extension on a namespace from another package. DocC strict-mode refused to resolve. Fix: prose paragraph with single-backtick `OTLP.TraceContext`. Reinforces existing memory entry [[feedback-docc-cross-package]]. Same finding re-surfaced for swift-log-otlp v0.3.
- **Hand-rolled JSON writer beats tree-construction-then-serialize.** swift-otlp-json's `JSONBuilder` (direct string emit + per-container `needsComma` stack) was simpler than going through swift-json's tree types. Pattern: for narrow-vocabulary canonical-output JSON, hand-roll. For arbitrary user-data round-trip, use swift-json. Memory note for future packages emitting fixed schemas.
- **Explicit-value APIs over implicit-context APIs in encoder layers.** As described in Item 2 reshape. Memory note: encoder packages must not depend on logging-frontend or tracing-frontend context bridges; adapter packages do that wiring.

None of these warrant a new cross-package RFC. **Documented absence of policy gaps. Criterion satisfied.**

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

Per-package review:

| Convention | swift-otlp-json | swift-tracing-otlp v0.3 | swift-log-otlp v0.3 |
|---|---|---|---|
| Module name PascalCase, no `Swift` prefix | ✓ OTLPJSON | ✓ TracingOTLP (unchanged) | ✓ LogOTLP (unchanged) |
| Apache-2.0 + LLVM exception SPDX | ✓ | ✓ | ✓ |
| NOTICE crediting upstream | ✓ OpenTelemetry opentelemetry-proto | ✓ (unchanged from v0.1) | ✓ (unchanged from v0.1) |
| Swift 6.0+ tools version | ✓ | ✓ | ✓ |
| macOS 14 platform floor | ✓ | ✓ | ✓ |
| Sendable-clean by default | ✓ | ✓ | ✓ |
| Strict concurrency in CI | ✓ | ✓ | ✓ |
| Single public error enum, typed throws | (no errors; non-throwing encoder) | ✓ TracingOTLPError (unchanged) | ✓ LogOTLPError (unchanged) |
| Public APIs Foundation-free | ✓ | ✓ | ✓ |
| Repo skeleton from `bare-swift new` | ✓ | ✓ (existing repo) | ✓ (existing repo) |
| README tagline + ≤30-line example | ✓ | ✓ updated with TraceContext example | ✓ updated with cross-signal example |
| CHANGELOG with v0.x entry per Keep-a-Changelog | ✓ | ✓ | ✓ |
| CI green on macOS + Linux | ✓ | ✓ | ✓ |
| DocC bundle | ✓ | ✓ | ✓ |
| docc-target set from initial commit (Gate 11 lesson) | ✓ | ✓ (unchanged) | ✓ (unchanged) |
| Sanitizers ON (no large static data) | ✓ | ✓ | ✓ |

### Deviations and findings

**1. Same-day three-tranche shipping cycle.** Phase 14 is the third single-day phase in the project (after Phase 11 and Phase 13). The pattern of "RFC accepted → all tranches shipped same day → Gate retrospective" is now canonical for small-LOC additive waves. Phases that need fresh implementations (Phase 10 brotli decoder, Phase 12 brotli encoder) still need multi-day spans.

**2. Hand-rolled vs library-based: a design pattern.** swift-otlp-json deliberately did **not** depend on swift-json. Reason: OTLP/HTTP/JSON has a fixed schema with proto3-mapping rules (64-bit ints as strings, NaN/Inf as strings, bytes as base64). A tree-construction phase would allocate twice; direct emit is faster + simpler. This is the **second** hand-rolled-vs-library decision in the project (swift-distributed-tracing-bridge chose hand-rolled Mutex over swift-atomics for its lookup table). Pattern: for fixed-schema or narrow-vocabulary outputs, hand-roll. For variable-schema / round-trip, depend on the library. Worth a memory note.

**3. Cross-package dependency graph deepens cleanly.** swift-otlp-json depends on **four** intra-ecosystem packages (swift-base64, swift-otlp-exporter, swift-tracing-otlp, swift-log-otlp). This is the deepest single-package dep graph in the ecosystem so far. SwiftPM resolved cleanly. CI matrix (Ubuntu 22.04/24.04 × Swift 6.0/6.1 + macOS 15 × Swift 6.0) all green on first push. The bare-swift dependency-graph hygiene (no version constraints other than `from:`, no `.exact`, no `.upToNextMajor` overrides) continues to scale.

**4. Skill-driven plan execution worked cleanly.** 14C used `superpowers:writing-plans` to author a 10-task plan in advance, then `superpowers:executing-plans` to walk the plan inline with TaskCreate/TaskUpdate tracking. Result: zero rework, zero failed steps. The skill workflow is now validated for small additive bumps; worth retaining as the default for v0.x+1 ships.

**5. Honest-scope shipping pattern reaffirmed (with a twist).** Unlike Phases 12 and 13 where the RFC overestimated and shipping scope-cut, Phase 14's actual LOC (~950) came in **under** the RFC estimate (~1000). This is the first phase where the estimate was accurate-to-conservative. Reason: the OTLP message types already existed in swift-tracing-otlp / swift-log-otlp / swift-otlp-exporter from Phases 3+4, so 14A's "encoder over existing types" was genuinely small.

### Verdict

RFC-0001 held cleanly under Phase 14 stress. Three tranches of varying shape (new encoder package, value-type addition to existing package, single-init convenience on existing package) all complied without exception.

---

## Item 5 — Phase 15 anchor decision ✗ NOT YET (recommendation)

Phase 15 anchor candidates surveyed (drawn from Gate 11–14 standing queue plus Phase 14's documented deferrals):

| Wave | Packages | Risk | Adoption fit |
|---|---|---|---|
| **swift-idna v0.2 (UTS #46 + NFC)** | swift-idna v0.2 | High (large Unicode tables + Bidi rule + ContextJ/O + NFC) | **High** — strongest queue candidate. Closes Phase 13's documented deferral. Unblocks swift-uri v0.3 auto-encode. Two-gate-deep standing winner since Phase 13. |
| swift-jwt-verify v0.2 (signing + RS256/ES384/EdDSA) | swift-jwt-verify v0.2 | Medium-high | Medium — closes verify-only design boundary; v0.1 has had ~2 days to settle. |
| swift-brotli v0.3 (streaming encoder) | swift-brotli v0.3 | Medium | Medium — narrower than IDNA. |
| OAuth 2.0 client primitives | swift-oauth2-client | Medium | Low-medium — no concrete adopter demand yet. |
| swift-distributed-tracing-otlp adapter | swift-distributed-tracing-otlp | Medium | Medium — would close Phase 14's "implicit context" gap. |
| Crypto-adjacent (swift-blake3, swift-siphash, swift-hkdf) | various | Medium-low | Low — 11th rejection candidate. |
| swift-publicsuffix v0.2 (ICANN/PRIVATE split) | swift-publicsuffix v0.2 | Low | Low-medium — closes a Phase 13 deferral; narrow scope. |

### Recommendation: **Anchor on swift-idna v0.2 (UTS #46 + NFC mapping)**

Reasons:

1. **Two-gate-deep consensus.** Phase 13 (Gate 13) deferred UTS #46 to a future phase. Phase 14 (Gate 14) named it as the natural next anchor in RFC-0019's "Sets up Phase 15+ options" section. The deferral is now aging; ship it.
2. **Closes a known gap blocking another package.** swift-uri v0.2's IDNA accessors work on already-ACE inputs only. swift-uri v0.3 auto-encode requires swift-idna v0.2's full UTS #46. Two packages are mutually waiting; Phase 15 unblocks both.
3. **Single-anchor phase shape.** Like Phase 10 (brotli decoder) and Phase 12 (brotli encoder), swift-idna v0.2 deserves a focused single-package phase rather than being one tranche of a multi-tranche wave. The honest-scope LOC will be substantial (estimated below).
4. **Audience is the same as Phase 13.** The internationalization adopters who used swift-publicsuffix and swift-idna v0.1.
5. **Risk profile is well-bounded but non-trivial.**
   - Unicode tables: IdnaMappingTable + NFC Canonical Combining Class + Canonical Decomposition tables. ~150 KiB packed. Same shape as swift-brotli's static dictionary (also ~150 KiB).
   - Algorithm: UTS #46 ToASCII / ToUnicode + NFC normalization. Well-specified; reference implementations in ICU, rust-idna, Python `idna`.
   - Bidi rule + ContextJ/O: per RFC 5891 § 4.2.3; well-specified.
   - The risk is **breadth** (many small specs to honor) not **depth** (no single hard algorithm).
6. **Sets up Phase 16+ options.** Once swift-idna v0.2 ships:
   - **swift-uri v0.3 auto-encode** becomes the natural Phase 16 single-tranche (~50 LOC additive on swift-uri).
   - **swift-jwt-verify v0.2 signing** stays available.
   - **swift-brotli v0.3 streaming** stays available.
   - **OAuth 2.0** stays available.

### Phase 15 tranche sketch (subject to formal RFC)

Single-anchor phase. Two reasonable shapes:

**Shape A (recommended): single tranche, honest-scope v0.2**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 15A | swift-idna v0.2 | UTS #46 mapping table + ToASCII (with bidi + ContextJ rules) + ToUnicode (basic). NFC normalization via embedded Canonical Decomposition table. **Defer:** ContextO (specialized rules), full Bidi disallowed-character set. | ~2 days |

**Shape B (alt): two tranches if scope-cut emerges**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 15A | swift-idna v0.2 | UTS #46 mapping table + ToASCII + NFC | ~1.5 days |
| 15B (stretch) | swift-uri v0.3 | Auto-encode hostnames via swift-idna v0.2 | ~0.5 day |

Total Phase 15 budget: 2-3 days wall-clock at observed pace.

### Why other waves were rejected

- **swift-jwt-verify v0.2 signing** — v0.1 (Phase 11) has only had 2 days to settle. Defer one more phase for adopter signal.
- **swift-brotli v0.3 streaming** — narrower audience than IDNA. Phase 16+ candidate.
- **swift-distributed-tracing-otlp adapter** — would close the "implicit context" gap from Phase 14 nicely, but no concrete demand from a downstream package yet. Wait for swift-distributed-tracing-bridge consumers to ask.
- **OAuth 2.0** — still no concrete adopter demand; premature.
- **swift-publicsuffix v0.2 (ICANN/PRIVATE split)** — narrow scope; better as a quiet patch when an adopter needs it.
- **Crypto-adjacent** — 11th consecutive rejection; swift-crypto stays entrenched.

**Action:** author the anchor decision as **RFC-0020** (Phase 15 anchor: swift-idna v0.2 UTS #46) and accept it before Phase 15 plans start.

---

## Open work to clear Gate 14

1. **Write RFC-0020** (Phase 15 anchor: swift-idna v0.2 UTS #46). 1-2 hour task. References this retrospective.
2. **Begin Phase 15 planning** after RFC-0020 accepts.

---

## What this retrospective changes for Phase 15

- The plan-then-execute-with-checkpoints workflow stays (validated cleanly on 14C via superpowers:writing-plans + executing-plans skills).
- **New procedural rule:** before drafting an RFC that names existing packages or version bumps, grep `packages/index.json` to verify package names and current versions. The Phase 14 doc-error pattern (3-for-3 across tranches) is preventable with a 30-second check. Worth a memory note.
- **New design pattern note:** encoder packages get explicit-input APIs; implicit-context wiring belongs in adapter packages. Phase 14 14B/14C reshape codified this.
- **Reinforced (not new):** Hand-rolled output beats tree-construction for fixed-schema outputs. Worth a memory note (second occurrence).
- **Reinforced (not new):** DocC cross-package symbol links don't resolve under strict mode. Use prose + single-backtick.
- **Carried forward:** Phase 14 left these documented gaps: automatic-context bridge (would be a swift-distributed-tracing-otlp adapter); structured-log-record bridges; JSON-Lines streaming. None block Phase 15; all are queued for Phase 16+.

---

## Decision

Gate 14 is **PASSED on items 1, 2, 3, 4** and **NOT YET PASSED on item 5 (open)**. Phase 15 plans should be drafted but not executed until item 5 closes via RFC-0020.

Phase 14 closed the longest-standing v0.x deferral in the observability tier — single-signal exporters since Phases 3+4 are now cross-signal-capable. The OTLP/HTTP/JSON transport adds a second wire format alongside protobuf. The W3C TraceContext propagation primitive is now first-class.

The next step is **RFC-0020 anchoring Phase 15 on swift-idna v0.2 UTS #46 + NFC**, which closes Phase 13's documented deferral and unblocks swift-uri v0.3 auto-encode. swift-jwt-verify v0.2 signing, swift-brotli v0.3 streaming, swift-distributed-tracing-otlp adapter, and OAuth 2.0 remain on the queue for Phase 16+.

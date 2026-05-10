# Gate 5 Retrospective: Phase 5 → Phase 6

**Date:** 2026-05-10
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 5 (the structured-data formats wave anchored by [RFC-0009](../../rfcs/0009-phase-5-anchor-structured-data-formats.md)) against the Gate 1/2/3/4 criteria template, and recommends whether Phase 6 should start. All four tranches (5A–5D) shipped, including the optional 5D stretch — second consecutive phase to ship its stretch (4D was the first). Two RFCs landed mid-phase (RFC-0009 anchor, RFC-0010 date/time policy), the first time non-anchor RFCs have moved during execution.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All anchored Tranche 5A/5B/5C/5D packages shipped at v0.1+, CI green, DocC live, listed | ✓ PASS |
| 2 | RFC-0009 commitments fulfilled (msgpack/cbor/toml + 5D stretch) | ✓ PASS |
| 3 | ≥1 new RFC accepted during Phase 5 | ✓ PASS — RFC-0010 |
| 4 | RFC-0001 conventions stress-test against Phase 5 packages | ✓ DONE BELOW |
| 5 | Phase 6 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0011 below) |

**Overall:** Items 1, 2, 3, 4 pass cleanly. Item 5 is open. Phase 6 plans should be drafted but not executed until item 5 closes via RFC-0011.

**Item 3 is the first full pass on this criterion in five gates.** Phases 2/3/4 each had only the anchor RFC accepted (which was authored at the *previous* gate's closeout). Phase 5 added RFC-0010 mid-flight when the same gap surfaced for the third time in two consecutive packages — proving the RFC process can capture cross-cutting decisions when they arise organically, not just at gate boundaries.

The roadmap's stop conditions did not trigger: Phase 5 calendar time was ~1.5 days (vs the 4–6 month original budget). No security incidents. Two same-day patches surfaced (swift-toml v0.1.0 → v0.1.1 for serializer order; swift-time pre-merge fix for an invalid-range trap) — both caught by consumer tests, neither shipped to users.

---

## Item 1 — Phase 5 packages shipped ✓ PASS

| Tranche | Package | v0.1+ tag | Tests | DocC | Index | CI |
|---|---|---|---|---|---|---|
| 5A | swift-msgpack | v0.1.0 | 43 | ✓ | ✓ | green |
| 5B | swift-cbor | v0.1.0 | 61 | ✓ | ✓ | green |
| 5C | swift-toml | v0.1.1 | 40 | ✓ | ✓ | green |
| 5D | swift-time | v0.1.0 | 38 | ✓ | ✓ | green |

All four tranches shipped. The 5D stretch was deliberately picked to close [RFC-0010](../../rfcs/0010-foundation-free-date-time-policy.md)'s date/time gap two phases sooner than the alternative (Phase 6 foundation tier).

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists 26 packages (1 from Phase 0, 10 from Phase 1, 5 from Phase 2, 2 from Phase 3, 4 from Phase 4, 4 from Phase 5).

**Calendar time:** Phase 5 executed in ~1.5 days. swift-cbor (5B) was the densest day with full RFC 8949 coverage including lossless float16-to-float32 decode and `-2^64..-1` negative-integer fidelity. swift-toml (5C) was the largest-LOC package but smaller than swift-uri's WHATWG state machine. swift-time (5D) was authored in parallel with the consumer test that found the only late-stage bug. 182 tests across the four packages, CI green on every merge, DocC live for every package.

---

## Item 2 — RFC-0009 commitments ✓ PASS

[RFC-0009](../../rfcs/0009-phase-5-anchor-structured-data-formats.md) committed to the structured-data formats wave with one stretch slot:

| RFC-0009 commitment | Outcome |
|---|---|
| Tranche 5A: swift-msgpack (MessagePack encoder + decoder) | ✓ shipped |
| Tranche 5B: swift-cbor (RFC 8949 CBOR encoder + decoder) | ✓ shipped |
| Tranche 5C: swift-toml (TOML 1.0 parser + serializer) | ✓ shipped |
| Tranche 5D *(stretch)*: pick from swift-jsonpath / swift-time / swift-yaml-lite | ✓ shipped — picked swift-time per RFC-0010 |
| Foundation-free, non-Codable across the wave | ✓ preserved across all four packages |
| Datetime values in 5A/5B/5C preserved as raw text per RFC-0010 (until swift-time ships) | ✓ then swift-time shipped, closing the gap |
| Phase 6 unblocked — JSON-tier or other waves can build on the format tier | ✓ format tier now has 9 packages |

The RFC's "out of scope for v0.1" list (Codable bridging, indefinite-length CBOR, canonical-form encoding, TOML 1.1, comment-preserving TOML round-trip) held in every package's v0.1 release.

Notable findings beyond the RFC's stated commitments:

- **First mid-phase RFC accepted** (RFC-0010). The five-incident date/time gap had been documented across Phases 3–5; landing the policy mid-Phase-5 unblocked the swift-time slot decision and wrote down the migration plan for the five affected packages.
- **First same-day patch release** (swift-toml v0.1.0 → v0.1.1). The bug — leaf tables not round-tripping inline — was caught by the consumer test, not the unit tests; fixed within an hour. Pattern: "consumer test is the integration test that test suites can miss".
- **First pre-merge bug catch from running `swift test` repeatedly.** swift-time's `digits..<9` range trap would only fire on inputs with > 9 fractional-second digits — exactly one test exercised this case. SIGTRAP from the swift-testing harness instead of the usual `Issue.record`. Fixed before merge.

---

## Item 3 — ≥1 new RFC accepted during Phase 5 ✓ PASS

**Pre-Phase 5 (Gate 4 closeout):** RFC-0009 (Phase 5 anchor) accepted 2026-05-09 to clear Gate 4 item 5.

**During Phase 5 execution:** **RFC-0010 (Foundation-free date / time policy) accepted 2026-05-10**, mid-flight, after swift-toml's datetime handling forced the question. RFC-0010 had been flagged in the Gate 4 retro as "medium priority parallel work"; it crystallized into a hard requirement when swift-toml v0.1 shipped with raw-string datetimes and the upcoming 5D stretch slot needed a target.

This is the first phase to land a non-anchor RFC during execution. The previous four gates all noted the same partial-pass pattern: "RFC process produces decisions at gate-closeout time, not mid-execution." RFC-0010 demonstrates the process can move faster when the trigger is a same-phase forcing function (here: 5D stretch slot decision had to be made), not a "nice-to-have" cleanup.

**Candidates surfaced during Phase 5 execution that warrant RFCs (next gate's work):**

### RFC-0011 — Phase 6 anchor [the gating RFC for closing Gate 5]

Same shape as RFC-0005 / RFC-0007 / RFC-0008 / RFC-0009. Decision space mapped in item 5 below.

### RFC-0012 *(deferred)* — Stdlib `Swift.Duration` interop

**Trigger:** swift-time defines its own `Duration` type rather than using stdlib `Swift.Duration`. RFC-0010 acknowledged this and noted "v0.2 may add a helper to convert between them." Not gating; defer until a real consumer asks.

### RFC-0011 (alternate) — GitHub Pages setup runbook

**Status:** Still deferred from Gate 4. Fix-on-first-failure pattern continues to work fine across all four Phase 5 repos. No change.

### Recommendation

Author and accept **RFC-0011** as the Phase 6 anchor. Skip the Pages runbook RFC — five new repos in two phases have all hit the same one-step fix successfully; an RFC for it would be ceremony, not value.

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

Per-package review against RFC-0001 for the 4 Phase 5 packages:

| Convention | All four | Notes |
|---|---|---|
| Module name PascalCase, no `Swift` prefix | ✓ | `MsgPack`, `CBOR`, `TOML`, `Time` |
| Apache-2.0 + LLVM exception SPDX | ✓ | Every source file. |
| NOTICE crediting upstream | ✓ | All four credit IETF specs (msgpack.org, RFC 8949, TOML 1.0, RFC 3339/RFC 1123); swift-time additionally credits Howard Hinnant's `daysFromCivil` and Sakamoto's weekday algorithm (both public-domain). |
| Swift 6.0+ tools version | ✓ | All four. |
| macOS platform floor | ✓ | All four at macOS 14. |
| Sendable-clean by default | ✓ | All public types `Sendable`. No `@unchecked` needed. |
| Strict concurrency in CI | ✓ | All four. |
| Single public error enum, typed throws | ✓ | `MsgPackError` (5 cases), `CBORError` (6), `TOMLError` (10 with line/column), `TimeError` (3). |
| Public APIs Foundation-free | ✓ | None expose `Data`, `URL`, `Date`, `Foundation.Calendar`. swift-time's `Calendar` is intentionally name-shared with `Foundation.Calendar` per RFC-0010 (consumers pick one). |
| Repo skeleton from `bare-swift new` | ✓ | All four. |
| Tests/Vectors/EXCEPTIONS.md | ✓ | All four. |
| README tagline + ≤30-line example | ✓ | All four. |
| CHANGELOG with v0.1 entry | ✓ | All four. |
| CI green on macOS + Linux | ✓ | All four. |
| DocC bundle | ✓ | All four. |

### Deviations and findings

**1. First pure-text-format parser (swift-toml) hit a serializer-order bug.**
- The TOML serializer emitted *every* nested table as `[section]` headers at the end of each level, which silently reordered entries when an inline table appeared before a scalar. Section headers redirect every following `key = value` line, so they have to come last — meaning leaf tables must round-trip *inline* at their original position. Caught by the consumer test (Cargo.toml-style fragment), not the unit tests.
- Fixed in v0.1.1 same-day. Recorded as `feedback_toml_serializer_order` memory.
- **Action:** none beyond the memory. Pattern applies to any text-format serializer with structural-delimiter scope (YAML, INI, etc.).

**2. Range trap in fractional-seconds parser caught pre-merge.**
- `for _ in digits..<9 where digits < 9 { n *= 10 }` constructs an invalid `Range` when `digits > 9`, traps with SIGTRAP. The `where` clause runs INSIDE the loop, after the range is built. Fix: explicit `if digits < 9 { for _ in digits..<9 { ... } }`.
- Caught by the "fractional seconds: more than 9 digits are truncated" test, but as a SIGTRAP rather than an `Issue.record` — surfaces only when running the test bundle directly, not via `swift test --filter`.
- **Action:** language-level paper cut; nothing to fix in the codebase. Worth remembering: `Range(a..<b) where a < b` is a runtime check, not a compile-time guard.

**3. swift-cbor's negative-integer fidelity required a design choice.**
- CBOR's negative integer range is `-1..-2^64`, exceeding `Int64.min`. Storing as `Int64` would lose `-2^64`; storing as `UInt64` wire-magnitude (where the actual integer is `-1 - storedValue`) preserves it.
- Decision: split signed (`uint(UInt64)`) and negative (`negative(UInt64)` storing wire magnitude). Same shape as MessagePack's split between `int` and `uint`. Pattern is now established for any future binary-format parser with > Int64 ranges.
- **Action:** documented in CBORValue's case-level docs.

**4. RFC-0010's "Calendar collides with Foundation.Calendar" warning is real.**
- Consumers using both `Time` and `Foundation` need to qualify with `Time.Calendar` to disambiguate. Documented; not a blocker for Foundation-free consumers (which is the intended audience).
- **Action:** none.

**5. Two-RFC-per-phase cadence.**
- Phase 5 is the first phase to land *two* RFCs (the anchor + an in-flight policy). Operationally, RFC-0010 took ~30 min to author once the trigger was clear. Future phases may follow this pattern when cross-cutting decisions surface mid-flight.
- **Action:** none. Watch for the same trigger shape (third-incident threshold) in Phase 6.

### Verdict

RFC-0001 held cleanly under Phase 5 stress (the largest text-format spec implemented to date — TOML 1.0 — plus the first foundation-tier package added since Phase 1). Conventions absorbed new patterns smoothly. The `Calendar` name collision with Foundation is documented and accepted; the negative-integer split pattern is now established for binary formats.

---

## Item 5 — Phase 6 anchor decision ✗ NOT YET (recommendation)

Phase 6 anchor candidates surveyed:

| Wave | Packages | Risk | Adoption fit |
|---|---|---|---|
| **JSON tier** | swift-json (RFC 8259 value type + parser + serializer), swift-jsonpath (RFC 9535), swift-jsonpatch (RFC 6902); swift-jsonschema-lite as 6D stretch | Medium | **High** — every HTTP API serializes JSON. Foundation-free + non-Codable + Bytes-aware fills a real gap. swift-jsonpointer (Phase 1) waits for a JSON value type to operate on. |
| Crypto-adjacent | swift-blake3, swift-siphash, swift-hkdf, swift-x25519 | Medium-low | Medium — swift-crypto is established; rejected for the same reasons in Phases 4 + 5 (third "rejected" is a real signal). |
| Concurrency primitives | swift-async-helpers, backpressure, channel | Medium | Low-medium — swift-async-algorithms exists with broad coverage. |
| Downstream v0.2 sweep | cookie / tracing-otlp / log-otlp / msgpack / cbor / toml minor bumps wiring in `Time` | Low | Closes RFC-0010's migration plan. Not a "wave" in the new-packages sense; better as a parallel work-stream during Phase 6. |
| Internationalization tier | swift-idna, swift-publicsuffix | Medium | Medium — swift-uri's deferred IDNA work would close. Niche but real. |

### Recommendation: **Anchor on the JSON tier**

Reasons:

1. **Audience continuity.** Phase 4's networking primitives (uri, mime, cookie) and Phase 5's structured-data formats (msgpack, cbor, toml) all imply JSON consumers. Every HTTP API serializes JSON; Foundation-free + non-Codable JSON is a real gap.
2. **Pairs with existing tier.** swift-jsonpointer (Phase 1) has been waiting for a JSON value type to operate on — currently consumers wire pointer evaluation to their own types. swift-jsonpath (RFC 9535) is a natural sibling. swift-jsonpatch (RFC 6902) closes the JSON-mutation triangle.
3. **Differentiation is real.** Foundation's `JSONSerialization` is Codable-backed, returns `Any`. swift-foundation has the same thing. A clean `JSONValue` enum + Bytes-in / String-in parser fills the niche the same way swift-bytes filled the buffer niche in Phase 2.
4. **Risk profile is known.** swift-json is moderate (RFC 8259 grammar; 1–2 days at observed pace). swift-jsonpath is small (path-expression compiler over a tree; ~1 day). swift-jsonpatch is small. swift-jsonschema-lite (the 6D stretch) is the wildcard; defer or scope narrowly.
5. **Sets up Phase 7+.** Once JSON exists: swift-otlp-exporter / swift-tracing-otlp / swift-log-otlp can ship JSON-mode encoding (long-deferred). swift-msgpack / swift-cbor can ship JSON ↔ binary helpers if asked. swift-jsonschema-lite becomes feasible.

### Phase 6 tranche sketch (subject to formal RFC)

| Tranche | Packages | Estimated calendar |
|---|---|---|
| 6A | swift-json (JSONValue + RFC 8259 parser + serializer) — the headline package | ~1–2 days at observed pace |
| 6B | swift-jsonpath (RFC 9535 over JSONValue) | ~1 day |
| 6C | swift-jsonpatch (RFC 6902 patch operations on JSONValue) | ~1 day |
| 6D *(stretch)* | swift-jsonschema-lite (subset of JSON Schema 2020-12) OR start the downstream v0.2 adoption sweep | ~1–2 days if any |

Plus, **in parallel with Phase 6 execution**, the **downstream v0.2 adoption sweep** committed in RFC-0010:

- swift-cookie v0.2 — `Cookie.expiresAt: Calendar?` alongside raw `expires`
- swift-tracing-otlp v0.2 / swift-log-otlp v0.2 — `Time.Instant` convenience initializers
- swift-msgpack v0.2 — `MsgPackValue.timestamp(Time.Instant)` for ext-type −1
- swift-cbor v0.2 — `CBOR.encodeDate(_:)` / `CBOR.decodeDate(...)` for tags 0/1
- swift-toml v0.2 — typed `TOMLValue.datetime(Time.Calendar)` accessor

Each is ~30 minutes of work (one new case / one new method, additive — no breaking changes). Six bumps in roughly half a tranche of effort.

Total Phase 6 budget: 1–2 weeks calendar at observed pace; 4–6 months at original roadmap pace.

### Why other waves were rejected

- **Crypto-adjacent** — third consecutive rejection. swift-crypto is too entrenched; differentiation requires niche positioning that doesn't fit the bare-swift audience.
- **Concurrency primitives** — second rejection. swift-async-algorithms exists with broad coverage; differentiation requires picking a specific gap (backpressure?) too narrow for an anchor.
- **Downstream v0.2 sweep alone** — not anchor-shape (no new packages, no new tier). Better as a parallel work-stream.
- **Internationalization tier** — niche; closes a real gap (swift-uri's IDNA deferral) but small audience compared to JSON.

**Action:** author the anchor decision as **RFC-0011** (Phase 6 anchor) and accept it before Phase 6 plans start. The downstream v0.2 sweep is committed in the same RFC as a parallel work-stream.

---

## Open work to clear Gate 5

1. **Write RFC-0011** (Phase 6 anchor: JSON tier + downstream v0.2 sweep). 1–2 hour task. References this retrospective.
2. **Begin Phase 6 planning** after RFC-0011 accepts.

**Recommended cadence:** RFC-0011 immediately after this retro is read → Phase 6 planning starts → first Tranche 6A package (swift-json) within a session.

---

## What this retrospective changes for Phase 6

- The plan-then-execute-with-checkpoints workflow stays. Phase 5 demonstrated it absorbs in-flight RFC authoring without breaking momentum.
- "Spec → plan → checkpointed inline execution" continues to scale; swift-time's parallel implementation alongside the consumer test caught bugs before merge.
- Inline-vector test pattern stays; RFC 8949 Appendix A canonical vectors validated swift-cbor.
- `docc-target` opt-in upfront stays canonical for any multi-target package.
- GitHub Pages two-step setup is now standard runbook for new repos. Five repos in Phases 4–5 hit the same one-step fix; will continue to apply ad-hoc rather than pre-create.
- Cross-package symbol references via single-backtick stays.
- **New:** mid-phase RFC authoring is an established pattern when a cross-cutting decision triggers a forcing function. Watch for the trigger shape (third-incident threshold) in Phase 6.
- **New:** the downstream-v0.2-sweep parallel work-stream is the migration vehicle for foundation-tier additions. Future RFCs that change shared types should follow the same pattern.
- **New:** consumer tests are the integration test the unit-test suite can miss. Run the consumer test before tagging any release.

---

## Decision

Gate 5 is **PASSED on items 1, 2, 3, 4** and **NOT YET PASSED on item 5 (open)**. Phase 6 plans should be drafted but not executed until item 5 closes via RFC-0011.

Phase 5 is the second consecutive phase to ship its 4D stretch and the first phase to land a non-anchor RFC during execution. The structured-data formats wave shipped in ~1.5 days with full RFC 8259 / RFC 8949 / TOML 1.0 coverage on the first three tranches and the foundation-tier date/time gap closed on the fourth. The format tier now has 9 packages; the foundation tier has 8.

The next step is **RFC-0011 anchoring Phase 6 on the JSON tier with the downstream v0.2 sweep as a parallel commitment**, which both clears Gate 5 and structures the next wave.

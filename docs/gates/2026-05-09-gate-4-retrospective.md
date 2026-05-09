# Gate 4 Retrospective: Phase 4 → Phase 5

**Date:** 2026-05-09
**Status:** Recommendation
**Author:** bare-swift project lead

This document evaluates Phase 4 (the networking-primitives wave anchored by [RFC-0008](../../rfcs/0008-phase-4-anchor-networking-primitives.md)) against the Gate 1/2/3 criteria template, and recommends whether Phase 5 should start. All four tranches (4A–4D) shipped, including the optional 4D stretch — the cleanest phase closeout to date.

---

## Verdict at a glance

| # | Criterion | Status |
|---|---|---|
| 1 | All anchored Tranche 4A/4B/4C/4D packages shipped at v0.1+, CI green, DocC live, listed | ✓ PASS |
| 2 | RFC-0008 commitments fulfilled (URL/MIME/cookie wave complete; 4D stretch shipped) | ✓ PASS |
| 3 | ≥1 new RFC accepted during Phase 4 | partial — see below |
| 4 | RFC-0001 conventions stress-test against Phase 4 packages | ✓ DONE BELOW |
| 5 | Phase 5 anchor decision recorded as RFC | ✗ NOT YET (recommendation: RFC-0009 below) |

**Overall:** Items 1, 2, 4 pass cleanly. Item 3 partially passes (RFC-0008 was accepted at the *start* of Phase 4 to satisfy Gate 3; no further RFCs landed during execution — same shape as Phase 2 / Phase 3). Item 5 is open. Phase 5 plans should be drafted but not executed until item 5 closes via RFC-0009.

The roadmap's stop conditions did not trigger: Phase 4 calendar time was ~1 day (vs the 4–6 month original budget). No security incidents. No Apple announcements that subsumed planned packages. RFC-0001 held under a *brand-new audience tier* (networking primitives — no upstream Rust crate to crib from on three of four packages).

---

## Item 1 — Phase 4 packages shipped ✓ PASS

| Tranche | Package | v0.1+ tag | Tests | DocC | Index | CI |
|---|---|---|---|---|---|---|
| 4A | swift-uri | v0.1.0 | 100 | ✓ | ✓ | green |
| 4B | swift-mime | v0.1.0 | 37 | ✓ | ✓ | green |
| 4C | swift-cookie | v0.1.0 | 35 | ✓ | ✓ | green |
| 4D | swift-log-otlp | v0.1.0 | 22 | ✓ | ✓ | green |

All four tranches shipped — including the 4D stretch, which RFC-0008 explicitly listed as optional. The 4D pick (swift-log-otlp) closed an outstanding deferral from Gate 3 ("ship next not 3c") and completes the OTLP signal triad alongside swift-otlp-exporter (metrics) and swift-tracing-otlp (traces).

The umbrella site at <https://bare-swift.github.io/bare-swift/> lists 22 packages (1 from Phase 0, 10 from Phase 1, 5 from Phase 2, 2 from Phase 3, 4 from Phase 4).

**Calendar time:** Phase 4 executed in ~1 day with all four tranches landed. swift-uri alone is the largest single-package effort in the ecosystem (~1,600 LOC source, ~1,500 LOC tests, 22-task plan executed inline) — completed in a single session. Test coverage is full (194 tests across the four packages), CI green on every merge, DocC live for every package, and end-to-end consumer integration verified for each.

---

## Item 2 — RFC-0008 commitments ✓ PASS

[RFC-0008](../../rfcs/0008-phase-4-anchor-networking-primitives.md) committed to the networking-primitives wave with one stretch slot:

| RFC-0008 commitment | Outcome |
|---|---|
| Tranche 4A: swift-uri (WHATWG URL parser + serializer) | ✓ shipped |
| Tranche 4B: swift-mime (MIME parser + IANA catalog) | ✓ shipped |
| Tranche 4C: swift-cookie (RFC 6265: parse + serialize) | ✓ shipped |
| Tranche 4D *(stretch)*: pick from swift-http-types-extras / swift-jsonpath / swift-log-otlp | ✓ shipped — picked swift-log-otlp |
| Foundation-free guarantee preserved (RFC-0001) | ✓ preserved across all four packages |
| Phase 5 unblocked — config/serialization parsers can take URL-parsed sources | ✓ swift-uri ships; URLs are now first-class inputs |

The RFC's "out of scope for v0.1" list (full IDNA, Public Suffix List, RFC 6570, base-URL relative resolution, file:// Windows drive letters, full WPT runner, RFC 6265bis edge cases, full IANA catalog) held in every package's v0.1 release.

Notable findings beyond the RFC's stated commitments:

- **swift-uri is the largest single-package effort in the ecosystem.** ~1,600 LOC source, ~1,500 LOC tests, full WHATWG state machine across ~30 states, IPv4/IPv6 parsers, 7 percent-encoding sets, ~60 hand-curated WPT round-trip vectors. Scaled cleanly under the "anchor → spec → plan → execute inline with checkpoints" pattern from Phase 3.
- **No upstream Rust crate to crib from on three of four packages.** swift-uri, swift-mime, swift-cookie all implement IETF specs directly. NOTICE files reflect this ("This is a native bare-swift package; it is not derived from any upstream Rust crate"). RFC-0001's "translated from Rust" framing remains valid for crypto/format work but does not bind networking primitives.
- **OTLP signal triad now closed.** swift-otlp-exporter (metrics) + swift-tracing-otlp (traces) + swift-log-otlp (logs). Same pattern reused: extends the shared `OTLP` namespace, depends on swift-otlp-exporter for common types, duplicates ProtoWriter/EncodeCommon internally per RFC-0007.

---

## Item 3 — ≥1 new RFC accepted during Phase 4 — partial

**Pre-Phase 4 (Gate 3 closeout):** RFC-0008 (Phase 4 anchor) accepted 2026-05-09 to clear Gate 3 item 5.

**During Phase 4 execution:** zero new RFCs accepted.

Same partial-pass pattern as Phase 2 and Phase 3. The RFC process produces decisions at gate-closeout time (anchoring the *next* phase) rather than mid-execution. This is operationally the right cadence — execution is mechanical once an anchor RFC exists; mid-execution RFCs would either be premature (no rough edges yet) or scope-creep.

**Candidates surfaced during Phase 4 execution that warrant RFCs (next gate's work):**

### RFC-0009 — Phase 5 anchor [the gating RFC for closing Gate 4]

Same shape as RFC-0005 / RFC-0007 / RFC-0008. Decision space mapped in item 5 below.

### RFC-0010 — Foundation-free date / time policy [medium priority]

**Trigger:** Phase 4 hit the missing-date-type gap *twice*:
- swift-cookie's `Expires` attribute is preserved as a raw string in v0.1; date parsing (RFC 1123 / asctime) deferred to v0.2 awaiting a Foundation-free date type.
- swift-distributed-tracing-bridge (Phase 3) introduced `TracerInstant` for span timing without any cross-package date primitive.

**Proposal:** Codify the policy. Either: (a) ship a `swift-time` foundation package with `Instant`, `Duration`, RFC 1123/3339 parsers, and the calendar minimum needed for HTTP cookies and distributed tracing; or (b) commit to per-package `String`/`UInt64` representations indefinitely with no shared date type. RFC would force the choice.

### RFC-0011 — GitHub Pages setup runbook [low priority]

**Trigger:** Two Phase 4 packages (swift-mime, swift-cookie, swift-log-otlp) hit the same first-release docs failure: new repos need GitHub Pages enabled AND the github-pages environment must allow `v*` tag deployments. Each repo paid the same one-time fix.

**Proposal:** Either pre-create the Pages environment in a repo template, or extend `bare-swift new` to call the GitHub APIs after `gh repo create`. RFC would scope the automation.

### Recommendation

Author and accept **RFC-0009** (Phase 5 anchor) as the next governance work. That alone clears the spirit of item 3.

RFC-0010 is a real gap and should be authored even if Phase 5 doesn't anchor on time — it informs Cookie v0.2 and any HTTP/2 work. RFC-0011 is a runbook nicety; defer or skip.

---

## Item 4 — RFC-0001 conventions stress-test ✓ DONE

Per-package review against RFC-0001 for the 4 Phase 4 packages:

| Convention | All four | Notes |
|---|---|---|
| Module name PascalCase, no `Swift` prefix | ✓ | `URI`, `MIME`, `Cookie`, `LogOTLP` |
| Apache-2.0 + LLVM exception SPDX | ✓ | Every source file. |
| NOTICE crediting upstream | ✓ | All four — swift-uri/mime/cookie credit IETF/IANA specs ("native bare-swift, no Rust crate"); swift-log-otlp credits opentelemetry-proto. |
| Swift 6.0+ tools version | ✓ | All four. |
| macOS platform floor | ✓ | All four at macOS 14 (none needed `Mutex`). |
| Sendable-clean by default | ✓ | All public types `Sendable`. No `@unchecked` needed. |
| Strict concurrency in CI | ✓ | All four. |
| Single public error enum, typed throws | ✓ | `URIError` (9 cases), `MIMEError` (5), `CookieError` (4), `LogOTLPError` (0 — extension point). |
| Public APIs Foundation-free | ✓ | None expose `Data`, `URL`, `Date`. |
| Repo skeleton from `bare-swift new` | ✓ | All four. |
| Tests/Vectors/EXCEPTIONS.md | ✓ | All four. |
| README tagline + ≤30-line example | ✓ | All four. |
| CHANGELOG with v0.1 entry | ✓ | All four. |
| CI green on macOS + Linux | ✓ | All four. |
| DocC bundle | ✓ | All four. |

### Deviations and findings

**1. swift-uri scaled the "spec → plan → checkpointed inline execution" pattern to ~1,600 LOC source.**
- 22-task plan executed inline with checkpoints after Tasks 1, 6, 9, 14, 17, 21. WHATWG state machine implemented region-by-region (basics → authority → special schemes → path/query/fragment) with the spec fetched directly via curl per region, not paraphrased through the plan.
- ~60 curated WPT round-trip vectors in `WPTRoundTripTests.swift`; surfaced three real bugs during integration (file:// missing `//`, trailing-slash dropped, special-scheme empty authority lost `//` marker). All three fixed before the first commit.
- **Action:** none. The pattern scales. Future large packages can follow the same shape.

**2. WHATWG percent-encode sets are NOT strictly nested.**
- Backtick is in fragment but NOT in query; returns in path. Initial implementation used cascading `if encoded for stricter set then fall through` logic and silently dropped backtick from path. Fixed by restructuring to flat per-set membership checks (a switch statement enumerating each set).
- Recorded as `feedback_swift_uri_lessons` memory.
- **Action:** none. Pattern applies to similar encode-set work in future format packages.

**3. SameSite.none / Optional.none assignment ambiguity.**
- `cookie.sameSite = .none` (where `sameSite` is `Cookie.SameSite?`) silently set the value to `nil` instead of the `.none` enum case. Compiler did not warn. Detected by a unit test.
- Recorded as `feedback_swift_samesite_none` memory. Fix is to qualify: `cookie.sameSite = Cookie.SameSite.none`.
- **Action:** memory captures it. Watch for similar trap in any enum with case named `none`/`some` stored in Optional.

**4. GitHub Pages setup on new bare-swift repos requires two API calls.**
- Pages must be enabled (`POST /repos/{owner}/{repo}/pages -f build_type=workflow`) AND the `github-pages` environment must allow `v*` tag deployments (`POST /repos/{owner}/{repo}/environments/github-pages/deployment-branch-policies -f name='v*' -f type='tag'`). The environment doesn't exist until first `docs.yml` deployment attempt, so the policy call must come after first failure.
- Hit on three Phase 4 packages (swift-mime, swift-cookie, swift-log-otlp). Pre-Phase 4 packages didn't hit it — likely a GitHub default change.
- Recorded as `feedback_github_pages_new_repo` memory.
- **Action:** RFC-0011 candidate (low priority). For now, fix-on-first-failure pattern is well understood.

**5. Cross-package DocC topic-list items reject single-backtick code spans.**
- swift-log-otlp's `## Topics` section listed `OTLP.encodeLogs(_:)` (cross-package symbol that can't be `` `` -wrapped per the DocC cross-package memory). Single-backtick (`` `OTLP.encodeLogs(_:)` ``) was reasonable but failed: DocC requires topic items to be valid links. Solution: move cross-package symbol references out of `## Topics` into prose paragraphs.
- Strengthens the existing `feedback_docc_cross_package` memory: cross-package symbols can appear in prose with single-backtick OR with full markdown link syntax, but cannot appear as `## Topics` items at all.
- **Action:** memory updated implicitly via this retrospective. Future multi-OTLP packages and similar cross-namespace work should put the entry-point reference in prose.

**6. Foundation-free date gap is now a confirmed two-incident pattern.**
- Phase 3: `swift-distributed-tracing-bridge` defined a custom `TracerInstant` for span timing (no shared date type to depend on).
- Phase 4: `swift-cookie` preserves `Expires` as a raw string and explicitly defers date parsing to v0.2 awaiting an ecosystem date type.
- Two incidents make it a real ecosystem gap. RFC-0010 should formalize the policy (ship `swift-time` foundation OR commit to per-package representations).
- **Action:** RFC-0010 candidate (medium priority; informs cookie v0.2).

### Verdict

RFC-0001 held cleanly under Phase 4 stress (the largest single-package effort to date; the first format-tier wave with no Rust upstream). The conventions absorbed new patterns smoothly. No conventions wrong; the date-type gap is the only genuinely-unaddressed RFC-shaped question after four phases.

---

## Item 5 — Phase 5 anchor decision ✗ NOT YET (recommendation)

Phase 5 anchor candidates surveyed:

| Wave | Packages | Risk | Adoption fit |
|---|---|---|---|
| **Structured-data formats** | swift-msgpack, swift-cbor, swift-toml, swift-jsonpath; YAML deferred | Medium | **High** — every server library deserializes config + protocol payloads. MessagePack and CBOR are well-bounded binary specs (~1 day each); TOML 1.0 is moderate (~2 days). |
| Time & dates foundation | swift-time (Instant, Duration, Calendar minimum, RFC 3339, RFC 1123) | Medium | High — fills two-incident ecosystem gap. Smaller wave (1 package + integrations); could be done as a side deliverable rather than an anchor. |
| Crypto-adjacent | swift-blake3, swift-siphash, swift-hkdf, swift-x25519 | Medium-low | Medium — swift-crypto is established. Differentiation requires careful positioning. Niche but real. |
| Concurrency primitives | swift-async-helpers (AsyncSequence utilities, backpressure) | Medium | Low-medium — swift-async-algorithms exists; differentiation is harder. Better fit as a tier 1+ side deliverable. |

### Recommendation: **Anchor on the structured-data formats wave**

Reasons:

1. **Audience continuity.** Phase 4's networking-primitives consumers overlap heavily with structured-data consumers — every HTTP API serializes JSON / MessagePack / Protobuf / TOML config / etc. swift-uri now exists; URL-as-config-source becomes natural with TOML/MessagePack as the deserializer.
2. **Foundation-free differentiation is real.** MessagePack and CBOR libraries in Swift today are typically Foundation-tied or use Codable heavily. A clean Bytes-in / Bytes-out implementation with no Foundation, no Codable dependency, fills a real gap.
3. **Risk profile is known.** MessagePack (RFC-less but well-spec'd) and CBOR (RFC 8949) are ~1 day each at observed pace. TOML 1.0 is moderate. JSONPath (RFC 9535) is small. YAML deliberately deferred — a single anchor wave shouldn't carry it.
4. **Builds on Phase 2 + Phase 4 foundation.** swift-bytes is the right type for binary parsers' input/output. swift-uri parses TOML's URL-shaped value types. swift-jsonpointer (Phase 1) pairs with swift-jsonpath as the format-tier query duo.
5. **Sets up downstream HTTP work.** Once MessagePack/CBOR exist, `Content-Type: application/msgpack` / `application/cbor` round-trip becomes natural with swift-mime; OTLP/json output becomes feasible (deferred from Phase 2/3/4).

### Phase 5 tranche sketch (subject to formal RFC)

| Tranche | Packages | Estimated calendar |
|---|---|---|
| 5A | swift-msgpack (MessagePack encoder + decoder; binary format spec) | ~1 day at observed pace |
| 5B | swift-cbor (RFC 8949 CBOR encoder + decoder) | ~1 day |
| 5C | swift-toml (TOML 1.0 parser + serializer) | ~2 days |
| 5D *(stretch)* | swift-jsonpath (RFC 9535) OR swift-time foundation OR swift-yaml-lite (a deliberately-narrowed YAML subset) | ~1 day if any |

Total Phase 5 budget: 1–2 weeks calendar at observed pace; 4–6 months at original roadmap pace.

The structured-data wave is rejected for Phase 2/3 ("each package competes with established alternatives") was correct *for those phases*. After Phase 4, the format tier exists with five packages (jsonpointer, dotenv, mime, cookie, uri); MessagePack/CBOR/TOML extend the tier rather than create it from nothing.

Time/dates is rejected as the *anchor* but should be authored as **RFC-0010** in parallel with RFC-0009 — it's a two-incident gap that informs cookie v0.2 and any HTTP/2 work. Could ship as a Phase 5 side deliverable or interleave with 5C/5D.

Crypto-adjacent is rejected for the same reason as Phase 4: swift-crypto is too entrenched, audience-fit is medium, not blocking adopters.

Concurrency primitives is rejected: swift-async-algorithms exists with good coverage; differentiation requires picking one specific gap (backpressure?) which is too narrow for an anchor.

**Action:** author the anchor decision as **RFC-0009** (Phase 5 anchor) and accept it before Phase 5 plans start. Optionally author **RFC-0010** (Foundation-free date / time policy) in parallel.

---

## Open work to clear Gate 4

1. **Write RFC-0009** (Phase 5 anchor: structured-data formats). 1–2 hour task. References this retrospective.
2. *(Recommended, parallel)* RFC-0010 (Foundation-free date / time policy). Decides whether Phase 5 ships swift-time or commits to per-package date representations.
3. *(Optional)* RFC-0011 (GitHub Pages setup runbook) — not blocking; defer or skip.
4. **Begin Phase 5 planning** after RFC-0009 accepts.

**Recommended cadence:** RFC-0009 immediately after this retro is read → Phase 5 planning starts → first Tranche 5A package within a session.

---

## What this retrospective changes for Phase 5

- The plan-then-execute-with-checkpoints workflow stays. Phase 4 demonstrated it scales to a 22-task ~1,600-LOC single-package effort (swift-uri).
- "Spec → plan → checkpointed inline execution" is now the canonical large-package pattern; `bare-swift new` + same-day-tag-and-release is the canonical small-package pattern.
- Inline-vector test pattern stays; ~60 curated WPT vectors for swift-uri proved the pattern handles spec-test subsets well.
- `docc-target` opt-in upfront stays canonical for any multi-target package.
- GitHub Pages two-step setup is now standard runbook for new repos (memory captures it).
- Cross-package symbol references via single-backtick stays; topic lists must use full DocC links or move to prose.
- Foundation-free date gap is recognized; RFC-0010 informs Phase 5 work even if Phase 5 doesn't anchor on it.
- The structured-data wave will pull `swift-bytes` and `swift-varint` heavily; both are battle-tested through three OTLP packages and one URL parser.

---

## Decision

Gate 4 is **PASSED on items 1, 2, 4** and **NOT YET PASSED on items 3 (partial) and 5 (open)**. Phase 5 plans should be drafted but not executed until item 5 closes via RFC-0009.

Phase 4 is the cleanest phase closeout to date. All four tranches shipped, including the optional 4D stretch that closed an outstanding deferral from Gate 3. The largest single-package effort in the ecosystem (swift-uri at ~1,600 LOC source) shipped in a single session with full test coverage and zero post-release bug fixes. The networking-primitives tier is now real: URL parsing, MIME types, cookies, and OTLP logs are all available Foundation-free in the bare-swift ecosystem.

The next step is **RFC-0009 anchoring Phase 5 on structured-data formats**, optionally accompanied by RFC-0010 (Foundation-free date / time policy), which both clears Gate 4 and structures the next wave.

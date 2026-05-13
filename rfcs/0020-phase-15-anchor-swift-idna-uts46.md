# RFC-0020 — Phase 15 anchor: swift-idna v0.2 (UTS #46 + NFC)

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-13 |
| Resolution | Accepted 2026-05-13 — Phase 15 plans may begin authoring. Single package: swift-idna v0.2. |

## Summary

Anchor Phase 15 on **swift-idna v0.2** — a single-anchor phase that upgrades swift-idna from its v0.1 Punycode-only codec to a full UTS #46 + NFC implementation. Structurally similar to Phase 10 (brotli decoder) and Phase 12 (brotli encoder): one focused package, substantial Unicode data tables, well-specified algorithm. Closes Phase 13's documented deferral. Unblocks the queued swift-uri v0.3 auto-encode work.

The wave is a single-tranche shape with an optional stretch tranche (15B) for swift-uri v0.3 auto-encode if 15A lands ahead of schedule.

## Problem

[Gate 14 retrospective](../docs/gates/2026-05-13-gate-14-retrospective.md) item 5 requires "Phase 15 anchor decision recorded as an RFC" before Phase 15 plans begin. The retrospective surveyed seven candidate waves (swift-idna v0.2 UTS #46, swift-jwt-verify v0.2 signing, swift-brotli v0.3 streaming, OAuth 2.0, swift-distributed-tracing-otlp adapter, crypto-adjacent, swift-publicsuffix v0.2); this RFC formalizes the choice.

**swift-idna v0.2 has been the standing-recommended Phase 15 candidate since Gate 13.** Phase 13 (Gate 13) deferred UTS #46 to a future phase because honest scope was ~3-5k LOC + ~200 KiB Unicode tables versus the optimistic original RFC-0018 estimate of ~800 LOC. Phase 14 (Gate 14) named it as the natural next anchor in RFC-0019's "Sets up Phase 15+ options" section. Two gates of consensus is enough.

**The concrete gap:** swift-idna v0.1 ships a working Punycode codec (RFC 3492). Callers must pre-normalize hostnames (lowercase + NFC) before calling `IDNA.toASCII(_:)`. Without UTS #46, three concrete things don't work:

1. **Mixed-case Unicode hostnames** — `Bücher.Example` doesn't round-trip; the caller has to lowercase first using a Unicode-aware case-folding (not just ASCII).
2. **Compatibility characters** — characters that UTS #46 maps to canonical equivalents (e.g., full-width ASCII, ligatures) aren't normalized; the caller must apply IdnaMappingTable manually.
3. **Disallowed-character rejection** — UTS #46 specifies a curated disallowed-character set (zero-width characters, control characters, etc.). swift-idna v0.1 has no such check; callers can pass through characters that real registries refuse.

**Downstream consequence:** swift-uri v0.2's IDNA accessors (`URI.Host.asciiDomain` / `URI.Host.unicodeDomain`) work on already-canonical inputs only. swift-uri v0.3 auto-encode (parse Unicode hostnames at URI-construction time) is blocked on swift-idna v0.2. Until swift-idna v0.2 lands, real-world adopters parsing arbitrary URLs with Unicode hostnames hit the v0.1 `URIError.idnaNotSupported` error.

After Phases 4 / 7 / 8 / 9 / 10 / 11 / 12 / 13 / 14, the bare-swift hostname-and-URL story is feature-complete except for this UTS #46 gap. Phase 15 closes it.

## Proposal

### Anchor: swift-idna v0.2 (single package)

**swift-idna v0.2** — UTS #46 ToASCII + ToUnicode + NFC normalization, replacing v0.1's caller-pre-normalize contract.

| API | Phase 13 v0.1 | Phase 15 v0.2 |
|---|---|---|
| `IDNA.toASCII(_:)` | Punycode encode; caller must pre-normalize | Full UTS #46 ToASCII (map → NFC → validate → Punycode) |
| `IDNA.toUnicode(_:)` | Punycode decode; caller gets raw decoded | Full UTS #46 ToUnicode (decode → validate → map back to display form) |
| `IDNA.process(_:)` | (new) | UTS #46 processing pipeline returning intermediate states; useful for inspection |
| `IDNAError` | 4 cases (RFC 3492 errors) | Adds: `disallowedCharacter`, `bidiViolation`, `contextJViolation`, `mappingNotFound`, `nfcNotRequiredButNotNFC` |

### Tranches

**Shape A (recommended): single anchor, honest-scope v0.2**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 15A | swift-idna v0.2 | UTS #46 mapping table + ToASCII + ToUnicode + NFC + Bidi rule + ContextJ | ~2 days |

**Shape B (alternate): two tranches if scope-cut emerges from 15A brainstorming**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 15A | swift-idna v0.2 | UTS #46 mapping table + ToASCII + ToUnicode + NFC. Defer Bidi + ContextJ to v0.3. | ~1.5 days |
| 15B (stretch) | swift-uri v0.3 | Auto-encode hostnames via swift-idna v0.2 | ~0.5 day |

The decision between A and B is deferred to the implementation plan. If 15A brainstorming surfaces a substantial scope-cut (precedent: brotli v0.2 encoder, idna v0.1 Punycode-only), Shape B becomes the fallback.

### Tranche 15A scope sketch

swift-idna v0.2 covers UTS #46 per the Unicode TR #46 specification:

- **IdnaMappingTable** (UTS #46 § 4 Table 1) — ~50 KiB packed. Maps each Unicode code point to one of: `valid`, `mapped` (with mapping), `deviation` (with mapping), `disallowed`, `disallowed_STD3_valid`, `disallowed_STD3_mapped`, `ignored`. Source: Unicode 16.0 (current at time of authoring).
- **NFC normalization** — Canonical_Combining_Class + Canonical_Decomposition tables. ~100 KiB packed. Apply NFC after IdnaMappingTable per UTS #46 § 4.1 step 4.
- **`IDNA.toASCII(_:)` pipeline:** Convert to Unicode → IdnaMappingTable → NFC → Bidi check (RFC 5893) → ContextJ check (RFC 5892 + RFC 5891 § 4.2.3) → Punycode-encode each label that contains non-ASCII → return ASCII result.
- **`IDNA.toUnicode(_:)` pipeline:** Punycode-decode each xn-- label → NFC → validate (same Bidi + ContextJ + IdnaMappingTable) → return Unicode result.
- **`IDNA.process(_:)`** — exposes the intermediate states (after mapping, after NFC, after validation, after encoding). Useful for inspection / debugging / custom pipelines.
- **`IDNAError`** typed-throws enum extended with the new UTS #46 error cases.

**Inputs handled correctly:**
- Mixed-case Unicode (`Bücher.Example` → `xn--bcher-kva.example`)
- Full-width ASCII (UTS #46 § 4 Table 1 `mapped` entries)
- Compatibility ligatures (`ﬃ` → `ffi`)
- Disallowed characters → typed-throws `IDNAError.disallowedCharacter`
- Bidi violations → typed-throws `IDNAError.bidiViolation`
- ContextJ violations (zero-width joiner / non-joiner usage rules) → typed-throws `IDNAError.contextJViolation`

**Out of scope for v0.2:**
- **ContextO** — the curated context-of rules for specific code points (RFC 5892 ContextO section). Smaller than ContextJ and rarely-hit in practice; defer to v0.3 if adopter demand surfaces.
- **STD3 ASCII rules toggle** — UTS #46 allows callers to opt into stricter ASCII validation. v0.2 hardcodes default (non-strict). Adopter knob deferred to v0.3.
- **Custom IdnaMappingTable** — UTS #46 § 4 mentions custom tables; v0.2 ships only the bundled Unicode 16.0 table. Custom-table support is a rare advanced use case.
- **TR #46 deviation handling toggle** — UTS #46 § 4 deviation characters have two valid behaviors (transitional vs nontransitional). v0.2 hardcodes nontransitional (the default; matches WHATWG URL spec). Transitional behavior deferred to v0.3.

### Tranche 15B scope sketch (stretch)

**swift-uri v0.3** adds auto-encode of Unicode hostnames at URI-construction time:

- swift-uri v0.2 added `URI.Host.asciiDomain` / `URI.Host.unicodeDomain` accessors operating on already-Host-typed values.
- swift-uri v0.3 wires swift-idna v0.2 into the `URI(string:)` parser: when the host portion contains non-ASCII characters, call `IDNA.toASCII(_:)` automatically.
- The v0.1/v0.2 `URIError.idnaNotSupported` case is replaced by `URIError.idnaProcessing(IDNAError)` carrying the underlying failure cause.
- ~50 LOC additive; all v0.2 tests still pass except the one that asserts `URIError.idnaNotSupported` for Unicode hostnames.

This is a minor breaking change to swift-uri (one URIError case rename + behavior change). v0.3 deserves its own version bump with a clear CHANGELOG migration note.

### Why this anchor

1. **Two-gate-deep consensus.** Phase 13 (Gate 13) deferred UTS #46. Phase 14 (Gate 14) named it as the natural next anchor. Two gates of consensus.
2. **Closes the longest-standing v0.x deferral in the URL/hostname tier.** swift-uri shipped v0.1 in Phase 4 (2026-05-10) with IDNA deferred. swift-uri v0.2 (Phase 13 13C) partially closed it (read-side accessors). v0.3 closes it fully — and v0.3 requires swift-idna v0.2.
3. **Audience continuity.** Same internationalization adopters from Phase 13 (PSL + IDNA Punycode users). URL parser adopters from Phase 4. The HTTP-server stack (Phase 11) and the observability stack (Phase 14) have both stabilized; the hostname tier needs equivalent care.
4. **Single-anchor shape works.** Phase 10 (brotli decoder, ~1700 LOC + 150 KiB static data) and Phase 12 (brotli encoder, ~1700 LOC) both proved that focused single-package phases can ship cleanly. swift-idna v0.2 is comparably-sized (estimated ~2500 LOC source + ~150 KiB Unicode tables).
5. **Risk profile is well-bounded.**
   - **Unicode tables:** UTS #46 IdnaMappingTable + NFC tables are well-specified; reference data files in the Unicode Consortium repo at <https://www.unicode.org/Public/idna/16.0.0/>.
   - **Algorithm:** UTS #46 is a small spec (10 pages) with a single processing pipeline. Reference implementations in rust-idna, Python `idna`, ICU.
   - **Bidi rule + ContextJ:** RFC 5893 + RFC 5892. Concrete rules over a small set of code points; testable with the UTS #46 test vectors.
   - **No new platforms / no new transports / no new wire formats.** This is a pure-data-and-algorithm package.
6. **Sets up Phase 16+ options.** Once swift-idna v0.2 + swift-uri v0.3 ship:
   - **swift-jwt-verify v0.2 signing** becomes the natural Phase 16 anchor (v0.1 will have had ~5-7 days to settle).
   - **swift-distributed-tracing-otlp adapter** stays available if swift-distributed-tracing-bridge consumers ask.
   - **swift-brotli v0.3 streaming** stays available.
   - **OAuth 2.0** stays available.

### Out of scope for Phase 15

- **swift-jwt-verify v0.2 signing.** Phase 16+ candidate.
- **swift-brotli v0.3 streaming.** Phase 16+ candidate.
- **swift-distributed-tracing-otlp adapter** (the "implicit context bridge" identified in Gate 14). No concrete demand yet.
- **OAuth 2.0 / crypto-adjacent.** Same standing rejections.
- **swift-publicsuffix v0.2 (ICANN/PRIVATE split).** Quiet patch when an adopter asks.
- **WHATWG URL parser feature parity.** swift-uri's WHATWG-divergence list is unchanged; Phase 15 is not a swift-uri compliance sweep.

### Tooling expectations

Phase 15 reuses all Phase 7–14 tooling without extension:

- swift-idna repo already exists from Phase 13.
- `bare-swift gen-site --umbrella .` after the v0.2 release.
- Strict-concurrency in CI stays mandatory.
- `docc-target` opt-in (already configured from Phase 13).
- **Sanitizers consideration:** swift-idna v0.2 will embed ~150 KiB of static Unicode tables. **Per Gate 10's lesson, sanitizers should be OFF for the test target** when static data exceeds 100 KiB. Phase 13 v0.1 had no large data; v0.2 will. CI workflow update required.
- Cross-library validation at ship time: round-trip Unicode hostname test vectors from the UTS #46 reference data through `IDNA.toASCII` and `IDNA.toUnicode`, verify outputs match the reference.
- Unicode-data regeneration script (`scripts/regenerate-idna-tables.sh`) — analogous to swift-publicsuffix's PSL regenerate script. Manual 1-2x/year cadence.

### Versioning

| Package | New | Future |
|---|---|---|
| swift-idna | v0.2 (UTS #46 + NFC + Bidi + ContextJ) | v0.3 = ContextO + STD3 toggle + transitional toggle if adopter demand |
| swift-uri (15B stretch) | v0.3 (auto-encode Unicode hostnames) | v0.4 = WHATWG-divergence sweep if demand surfaces |

Phase 15 introduces **0 new packages** (swift-idna repo exists from Phase 13). 1 minor version bump (15A) plus an optional stretch tranche (15B). Ecosystem stays at **46 packages**.

### Anti-goals

- **No new packages.** Phase 15 is an internal v0.x upgrade.
- **No transitional / nontransitional toggle.** Hardcode nontransitional (WHATWG default).
- **No custom IdnaMappingTable support.** Bundle the Unicode 16.0 table; that's it.
- **No retroactive bug-fix sweep across prior phases.**
- **No swift-uri WHATWG compliance push.** 15B (if shipped) is auto-encode only; nothing else changes in swift-uri.

## Alternatives considered

### Anchor Phase 15 on swift-jwt-verify v0.2 (signing + RS256/ES384/EdDSA)

**Pros:** closes the verify-only design boundary; rounds out the JWT story for issuers.

**Cons:** v0.1 (Phase 11) shipped 2026-05-12 — only 2 days ago. Adopter signal from real verifier usage hasn't accumulated. Phase 16+ is the right time.

**Verdict:** rejected as Phase 15. Strong Phase 16 candidate.

### Anchor Phase 15 on swift-brotli v0.3 (streaming encoder)

**Pros:** closes the v0.2 one-shot-only deferral; streaming is useful for large HTTP responses.

**Cons:** narrower audience than IDNA. swift-brotli v0.2 shipped 2026-05-13 (today) and won't have hit size limits in production yet.

**Verdict:** rejected as Phase 15. Phase 16+ candidate.

### Anchor Phase 15 on swift-distributed-tracing-otlp adapter

**Pros:** would close the "implicit context" gap identified in Gate 14 (where 14B/14C reshaped from RFC-0019's automatic-context-reading sketch to explicit-input APIs).

**Cons:** no concrete demand from a downstream package or adopter yet. The gap is a design observation, not a blocker. Wait for swift-distributed-tracing-bridge consumers to ask.

**Verdict:** rejected as Phase 15. Phase 16+ candidate if demand surfaces.

### Anchor Phase 15 on OAuth 2.0 client primitives

**Pros:** composes naturally on top of swift-bearer + swift-jwt-verify.

**Cons:** still no concrete adopter demand. OAuth 2.0 has many flows; picking the right v0.1 subset needs adopter input.

**Verdict:** rejected as Phase 15. Defer until concrete demand.

### Skip ahead, do crypto-adjacent (swift-blake3 / swift-siphash / swift-hkdf)

**Pros:** would complement swift-jwt-verify if signing ever lands.

**Cons:** swift-crypto remains entrenched. **11th consecutive rejection** across gates. Differentiation hasn't strengthened.

**Verdict:** rejected.

### Anchor Phase 15 on swift-publicsuffix v0.2 (ICANN/PRIVATE split)

**Pros:** closes a quiet Phase 13 deferral; small scope.

**Cons:** narrow. Better as a v0.1.1 patch when an adopter needs it. A full Phase 15 anchor would be over-rotating on a small feature.

**Verdict:** rejected as Phase 15 anchor. Will land as a quiet patch when demand surfaces.

## Resolution

**Accepted 2026-05-13.** Phase 15 plans may begin authoring. Single package: swift-idna v0.2.

Phase 15 closes the longest-standing v0.x deferral in the URL/hostname tier (swift-uri's Phase-4 IDNA deferral is now in its 15th phase). After Phase 15, the bare-swift URL parsing story supports full Unicode hostnames end-to-end: `URI(string: "https://Bücher.Example/")` parses, normalizes per UTS #46, and yields a Host with both `.asciiDomain` (`xn--bcher-kva.example`) and `.unicodeDomain` (`bücher.example`) accessors.

swift-jwt-verify v0.2 signing, swift-brotli v0.3 streaming, swift-distributed-tracing-otlp adapter, and OAuth 2.0 remain on the queue for Phase 16+.

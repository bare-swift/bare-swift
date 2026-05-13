# RFC-0018 — Phase 13 anchor: Internationalization wave (PSL + IDNA)

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-13 |
| Resolution | Accepted 2026-05-13 — Phase 13 plans may begin authoring. First package: swift-publicsuffix. |

## Summary

Anchor Phase 13 on the **internationalization wave**: ship swift-publicsuffix (Mozilla PSL longest-match lookup) and swift-idna (UTS #46 + Punycode per RFC 3492). Two medium packages, structurally similar to Phase 11's auth-primitives wave. Closes swift-uri's IDNA deferral from Phase 4 — the longest-standing v0.1 limitation in the URI family.

The wave decomposes into independent tranches: 13A swift-publicsuffix, 13B swift-idna, 13C optional swift-uri v0.2 integration.

## Problem

[Gate 12 retrospective](../docs/gates/2026-05-13-gate-12-retrospective.md) item 5 requires "Phase 13 anchor decision recorded as an RFC" before Phase 13 plans begin. The retrospective surveyed five candidate waves (internationalization, OTLP cross-signal v0.2, swift-jwt-verify v0.2 signing, OAuth 2.0, crypto-adjacent); this RFC formalizes the choice.

Internationalization has been the standing-recommended Phase 13 candidate since Gate 11's retro (and was suggested as Phase 12 alternate). Three consecutive gate retrospectives have considered i18n and haven't found a reason to defer further.

**The concrete gap:** swift-uri shipped in Phase 4 with explicit IDNA deferral. Hostname comparison in HTTPS clients — TLS certificate matching, CORS origin validation, cookie domain matching — all require IDNA-correct comparison across Unicode normalization forms. Without it, swift-uri users can't safely compare hostnames like `bücher.example` (which has multiple legal UTF-8 / NFC / NFKC encodings) against equivalent Punycode (`xn--bcher-kva.example`).

After Phases 4 / 7 / 8 / 9 / 10 / 11 / 12, the bare-swift HTTP stack is feature-complete except for this IDNA gap. Phase 13 closes it.

## Proposal

### Anchor: internationalization wave (2 packages + 1 optional integration)

- **swift-publicsuffix (Mozilla PSL longest-match)** — parse the Public Suffix List into a lookup structure and answer `(host) → registrable-domain`. ~400 LOC + ~150 KiB embedded data. Smallest, ships first.
- **swift-idna (UTS #46 + Punycode RFC 3492)** — convert IDN hostnames between Unicode (NFC) and ACE (Punycode) form per UTS #46 transitional and non-transitional rules. ~800 LOC + ~50 KiB data (IDNA mapping table). Hardest, ships second.
- **swift-uri v0.2 *(stretch)*** — adapt swift-uri's hostname-normalization path to call swift-idna. ~50 LOC + tests covering Unicode hosts.

Total source: **~1250 LOC + ~200 KiB data** across the wave — comparable in shape to Phase 11 (auth primitives, ~700 LOC) and Phase 10 (single brotli + content-encoding, ~1900 LOC). Calendar budget: ~3-4 days wall-clock.

### Tranches

| Tranche | Package | Estimated calendar |
|---|---|---|
| 13A | swift-publicsuffix — Mozilla PSL longest-match | ~1 day |
| 13B | swift-idna — UTS #46 + RFC 3492 Punycode | ~2-3 days |
| 13C *(stretch)* | swift-uri v0.2 — wire swift-idna for hostname normalization | ~0.5 day |

Total Phase 13 budget: ~3-4 days. Resembles Phase 7 / Phase 11 in shape.

### Why this anchor

1. **Closes the longest-standing API gap.** swift-uri shipped with explicit IDNA deferral in Phase 4 (2026-05-08). Five gates later, the deferral has aged the most of any deferred feature.
2. **Audience continuity is total.** Same HTTP server/client implementers Phases 4 / 7 / 8 / 9 / 10 / 11 / 12 serve. PSL + IDNA finish the URI story.
3. **Decomposes naturally.** swift-publicsuffix is data-heavy + algorithmically simple. swift-idna is data-medium + algorithmically heavy. Independent shipping order; each package is usable on its own.
4. **Two-gate-deep consensus.** Gate 11 + Gate 12 both name internationalization as the strongest next signal.
5. **Risk profile is well-bounded.**
   - swift-publicsuffix: longest-match suffix lookup over the Mozilla PSL. Algorithm is one page. The work is in the data ingestion script + license attribution.
   - swift-idna: UTS #46 has clear test vectors (W3C IDNA-table-tests). Punycode (RFC 3492) is a small algorithm with worked-example test vectors in the RFC. ICU's IDNA path is the oracle for cross-validation.
   - swift-uri v0.2 (if 13C ships): a thin adapter layer; risk is in test surface, not LOC.
6. **Data-heavy package precedent already exists.** swift-brotli (Phase 10) established the "MIT-licensed data + clean-room code" template (122 KiB dictionary). swift-publicsuffix reuses the same NOTICE pattern + `scripts/` generation + sanitizer-off CI config (for the 150 KiB PSL static array).
7. **Sets up Phase 14+ options.** Once i18n ships:
   - **OTLP cross-signal v0.2** becomes the natural Phase 14 anchor.
   - **swift-jwt-verify v0.2 signing + RS256** stays available.
   - **OAuth 2.0 client primitives** stay available.
   - **swift-brotli v0.3 streaming encoder** stays available.

### Tranche 13A scope sketch

`swift-publicsuffix` v0.1 covers Mozilla PSL longest-match lookup:

- `PublicSuffix.registrableDomain(of host: String) -> String?` — given a hostname, return the longest registrable domain (e.g., `"foo.bar.example.co.uk"` → `"bar.example.co.uk"`). Returns nil for hostnames that don't have a valid registrable domain (e.g., `"localhost"`, `"co.uk"` itself).
- `PublicSuffix.publicSuffix(of host: String) -> String?` — return just the public suffix portion (`"example.co.uk"` → `"co.uk"`).
- `PublicSuffix.isPublicSuffix(_ candidate: String) -> Bool` — convenience query.
- Embedded PSL: ~150 KiB static `[UInt8]` (LSV2 text + index built at compile time). Generated via `scripts/regenerate-psl.sh` (manual; PSL updates 1-2x/year).
- Foundation-free public API. swift-bytes is the only dep.
- License attribution: Mozilla Public License 2.0 for the PSL data; clean-room Swift for the lookup logic. NOTICE clearly separates.

**Out of scope for v0.1:**
- Automated PSL update workflow. v0.2+ if there's demand.
- ICANN vs private suffix differentiation. v0.1 treats all PSL entries uniformly. v0.2 may add a `(suffix, kind)` API.
- Wildcard rules (`*.foo.example`) AND exception rules (`!exception.foo.example`) per PSL § 3 — both implemented.

### Tranche 13B scope sketch

`swift-idna` v0.1 covers UTS #46 + RFC 3492 Punycode:

- `IDNA.toASCII(_ unicodeHost: String, transitional: Bool = false) throws(IDNAError) -> String` — convert Unicode hostname to ACE form (`bücher.example` → `xn--bcher-kva.example`).
- `IDNA.toUnicode(_ aceHost: String) throws(IDNAError) -> String` — convert ACE form back to Unicode.
- `IDNA.compareEqual(_ a: String, _ b: String) -> Bool` — IDNA-correct hostname equality (any combination of Unicode/ACE; case-insensitive per UTS #46 mapping).
- Punycode (RFC 3492) inner codec: `Punycode.encode([UInt32]) -> String` and `Punycode.decode(String) throws(PunycodeError) -> [UInt32]`.
- Embedded IDNA mapping table: ~50 KiB static (UTS #46 character classes: disallowed / mapped / deviation / valid / ignored). Generated from Unicode CLDR.
- Foundation-free public API. Depends on swift-publicsuffix? Probably not directly — IDNA is per-label, PSL is per-host. Caller composes them.

**Out of scope for v0.1:**
- Bidi (UTS #46 § 4.1 Bidi rule) — defer to v0.2 if there's demand.
- ContextJ / ContextO rules (UTS #46 § 4.2) — defer to v0.2.
- IDNA 2003 compatibility mode — v0.1 is IDNA 2008 + UTS #46 only.

### Tranche 13C scope sketch (stretch)

`swift-uri` v0.2 wires swift-idna into hostname normalization:

- `URI.normalizedHost: String` — apply IDNA toUnicode + lowercase per RFC 3986 § 6.2.3.
- `URI.host` (existing) unchanged — caller opts into normalization.
- Test surface: round-trip Unicode hosts; compare URLs with Punycode vs Unicode hosts and verify equality after normalization.
- Adds swift-idna as a dep.

**Out of scope:** changing existing v0.1 behavior. v0.2 is purely additive.

### Out of scope for Phase 13

- **DNS resolution.** Hostname → IP is a different concern; lives in swift-nio territory.
- **TLS hostname verification.** Composition over swift-idna + swift-nio.
- **HTTP-specific i18n (RFC 3987 IRIs).** Different spec; v0.2 of swift-uri if adopter demand surfaces.
- **OTLP v0.2 cross-signal.** Phase 14 anchor candidate.
- **OAuth 2.0 / swift-jwt-verify v0.2.** Phase 14+ candidates.

### Tooling expectations

Phase 13 reuses all Phase 7–12 tooling plus the data-heavy template:

- `bare-swift new` for the two new package scaffolds.
- `bare-swift gen-site --umbrella .` after each v0.1 release. Ecosystem grows from **43 → 45 packages**.
- Pre-emptive Pages tag-policy stays canonical.
- Strict-concurrency in CI stays mandatory.
- `docc-target` opt-in from scaffold time (mandatory from Gate 11 lesson).
- **TSan sanitizer off for swift-publicsuffix** (150 KiB static data triggers the same TSan BUS-fault as swift-brotli's 122 KiB dictionary — Phase 10 lesson). Other packages keep sanitizers enabled.
- Cross-library validation at ship time, NOT in test target (Foundation-free invariant from Gate 8/9 standing memory). For swift-idna: validate against ICU's IDNA path via a Foundation-using consumer script.

### Versioning

Two new packages get v0.1.0 tags; one existing gets a v0.2.0 bump:

| Package | New | Future |
|---|---|---|
| swift-publicsuffix | v0.1 (Mozilla PSL longest-match) | v0.2 if PSL automation or ICANN/private differentiation surface |
| swift-idna | v0.1 (UTS #46 + Punycode RFC 3492) | v0.2 = Bidi + ContextJ/O |
| swift-uri | v0.2 (stretch — wire swift-idna) | v0.3 for IRI support if demand |

Phase 13 introduces **2 new packages**, taking the ecosystem from **43 to 45 packages**.

### Anti-goals

- **No new tier on the umbrella.** Both packages join existing tiers: swift-publicsuffix in "format" or new "internet" sub-tier (TBD by `gen-site` tier-grouping). swift-idna similarly.
- **No automated PSL/Unicode update workflow.** v0.1 is manual regeneration.
- **No DNS or TLS integration.** Composition target, not Phase 13 scope.
- **No retroactive bug-fix sweep across prior phases.**

## Alternatives considered

### Anchor Phase 13 on OTLP cross-signal v0.2

**Pros:** closes Phase 2/3/4 single-signal deferral.

**Cons:** narrower audience than i18n. Three packages of medium complexity, each requiring protobuf encoder updates. The IDNA deferral is older + more impactful.

**Verdict:** rejected as Phase 13. Strong Phase 14 candidate.

### Anchor Phase 13 on swift-jwt-verify v0.2 (signing + RS256)

**Pros:** complements the just-shipped v0.1; closes the verify-only design boundary.

**Cons:** swift-jwt-verify v0.1 shipped in Phase 11 (very recent). Let v0.1 settle and gather adopter signal first. Single-package phase shape is fine but the demand signal isn't there yet.

**Verdict:** rejected as Phase 13. v0.2 minor release if adopter demand surfaces; otherwise wait.

### Anchor Phase 13 on OAuth 2.0 client primitives

**Pros:** sits naturally atop swift-bearer; closes the auth composition story.

**Cons:** no concrete adopter demand yet. OAuth 2.0 has many flows; picking the right v0.1 subset needs adopter input. Premature.

**Verdict:** rejected as Phase 13. Defer until concrete demand.

### Skip ahead, do crypto-adjacent (swift-blake3 / swift-siphash / swift-hkdf)

**Pros:** would complement swift-jwt-verify if signing eventually lands.

**Cons:** swift-crypto remains entrenched. Tenth rejection candidate across gates. Differentiation hasn't strengthened.

**Verdict:** rejected.

## Resolution

**Accepted 2026-05-13.** Phase 13 plans may begin authoring. First package: swift-publicsuffix.

Phase 13 closes the longest-standing API gap in the ecosystem (swift-uri's IDNA deferral from Phase 4) by adding PSL longest-match + UTS #46 IDNA primitives. After Phase 13, the bare-swift URI / hostname story is complete: parse + serialize URIs (Phase 4 swift-uri), normalize hostnames across Unicode + Punycode (Phase 13 swift-idna), classify hostnames by public-suffix (Phase 13 swift-publicsuffix).

OTLP cross-signal, swift-jwt-verify v0.2 signing, OAuth 2.0, and swift-brotli v0.3 streaming remain on the queue for Phase 14+.

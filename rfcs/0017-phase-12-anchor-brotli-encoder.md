# RFC-0017 — Phase 12 anchor: swift-brotli v0.2 encoder

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-12 |
| Resolution | Accepted 2026-05-12 — Phase 12 plans may begin authoring. First package: swift-brotli v0.2.0. |

## Summary

Anchor Phase 12 on **swift-brotli v0.2 — the encoder**. Single-anchor phase, mirroring Phase 10's shape (single anchor + small follow-on). Closes the longest-standing v0.2 deferral in the compression tier and completes the symmetric request/response compression story across all five web codings (identity / gzip / deflate / br).

Optional follow-on: **swift-content-encoding v0.4** adds the `br` encode branch to its multiplexer once the encoder ships.

## Problem

[Gate 11 retrospective](../docs/gates/2026-05-12-gate-11-retrospective.md) item 5 requires "Phase 12 anchor decision recorded as an RFC" before Phase 12 plans begin. The retrospective surveyed six candidate waves (swift-brotli v0.2, internationalization, OTLP cross-signal, swift-jwt-verify v0.2 signing, OAuth 2.0 client primitives, crypto-adjacent); this RFC formalizes the choice.

Phase 10 anchored on swift-brotli v0.1 decoder with explicit decoder-first staging — v0.2 encoder was named as the symmetric follow-on. Phase 9 established the pattern (deflate / gzip / zlib v0.2 encoders after v0.1 decoders); applying it to brotli is the natural completion of the compression tier.

Gate 9, Gate 10, and Gate 11 each named brotli v0.2 as the strongest forward signal:
- Gate 9 listed it as a viable Phase 10 candidate.
- Gate 10 named it as a strong Phase 11 alternative if auth fell through.
- Gate 11 (just closed) recommends it as the natural Phase 12 anchor.

Three consecutive gate recommendations is a strong-enough signal to commit.

## Proposal

### Anchor: swift-brotli v0.2 encoder

Phase 12 primary work is **swift-brotli v0.2** — the brotli compressor matching the v0.1 decoder we shipped in Phase 10.

**v0.2 scope:**

- `Brotli.compress(_ input: [UInt8], quality: Quality = .default) throws(BrotliError) -> [UInt8]` (or `Bytes`-based equivalent matching the v0.1 decoder's API shape).
- `Quality` enum exposing the canonical brotli quality tiers (`0`–`11` per RFC 7932; map to ergonomic Swift cases like `.fastest`, `.balanced`, `.smallest`, plus explicit `.level(Int)` escape hatch).
- Sliding-window management, ring buffer, literal/match/copy decision logic per RFC 7932 § 9.
- Round-trip tests against the v0.1 decoder (encoder output must decode back to the input).
- Cross-validation against `bro`/`brotli` CLI test vectors at ship time (NOT in the test target — Foundation-free invariant).

**Out of scope for v0.2:**
- Streaming encoder API. v0.2 is one-shot `[UInt8] in → [UInt8] out`. v0.3 may add a streaming/chunked interface if adopter demand surfaces.
- Custom shared-dictionary support. v0.2 uses only the RFC 7932 static dictionary (already embedded for the decoder).
- Quality-tuning heuristics beyond the canonical 0–11 levels.

The encoder is **~1500 LOC source estimate**. Brotli encoding is more complex than decoding (the decoder follows a deterministic state machine; the encoder makes literal/match/copy choices that affect compression ratio). But correctness has a clean check: round-trip against the v0.1 decoder we already shipped.

### Tranches

| Tranche | Package | Estimated calendar |
|---|---|---|
| 12A | swift-brotli v0.2 (encoder) | ~2-3 days |
| 12B *(stretch)* | swift-content-encoding v0.4 (add `br` encode branch) | ~0.5 day |

Total Phase 12 budget: ~3 days wall-clock at observed pace. Resembles Phase 10 (single anchor + small follow-on).

### Why this anchor

1. **Closes Phase 10's decoder-first staging explicitly.** Phase 10 RFC named v0.2 encoder as the future follow-on; this RFC fulfills it.
2. **Closes the last asymmetry in the compression tier.** After Phase 9, the deflate / gzip / zlib codecs had v0.2 encoders; only brotli still had decoder-only. After Phase 12, all four codecs support both directions.
3. **Audience continuity is total.** Same HTTP server/client implementers Phases 4 / 7 / 8 / 9 / 10 / 11 serve. Encoder enables response compression with brotli — the highest-compression web coding — and rounds out the `Content-Encoding` story for outbound traffic.
4. **Single-anchor phase shape works.** Phase 10 (swift-brotli v0.1 + content-encoding v0.3) shipped in ~1 day with zero coordination friction. Phase 12 mirrors that shape.
5. **Risk profile is well-bounded.**
   - The decoder we already shipped provides ground truth via round-trip vectors.
   - The static dictionary, transforms, and LUTs already exist in the v0.1 codebase — no new data work needed.
   - The encoder doesn't pull new deps; swift-brotli's only dep stays swift-bytes 0.1.0.
   - Quality tuning is bounded by RFC 7932's 0–11 levels.
6. **Sets up Phase 13+ options.** Once brotli encoder ships:
   - **Internationalization** (IDNA / PSL) becomes the natural Phase 13 anchor. It has data-heavy precedent established by swift-brotli (122 KiB dictionary).
   - **OTLP cross-signal v0.2** stays available.
   - **swift-jwt-verify v0.2** (signing + RS256) stays available.
   - **OAuth 2.0 client primitives** stay available.

### Tranche 12A scope sketch

`swift-brotli` v0.2 adds the encoder alongside the v0.1 decoder:

- New module-level entry point: `Brotli.compress(_:quality:) throws(BrotliError) -> [UInt8]` (exact name per the existing v0.1 `Brotli.decompress` shape).
- New `Brotli.Quality` enum: `.fastest` (quality 0), `.fast` (quality 4), `.default` (quality 6), `.balanced` (quality 9), `.smallest` (quality 11), plus `.level(Int)` for explicit control.
- Internal types: ring buffer, hash table for backreference search, literal/distance context model.
- `BrotliError` gains 1–3 encoder-specific cases (e.g. `.inputTooLarge`, `.qualityOutOfRange` if `.level(_)` carries an invalid value). Existing decoder error cases unchanged.
- Tests: round-trip vectors (encode → decode → assert ==), plus deterministic-output checks for fixed inputs at each quality tier.

**Out of scope for v0.2:**
- Streaming API.
- Custom dictionaries.
- Heuristic tuning beyond the RFC 7932 quality knob.

### Tranche 12B scope sketch (stretch)

`swift-content-encoding` v0.4 wires the brotli encoder into the response-encoding multiplexer:

- Existing v0.3 decode multiplexer already supports `br` (added when swift-brotli v0.1 shipped).
- v0.4 adds `br` to the encode side. Existing encode multiplexer supports `identity` / `gzip` / `deflate`; adding `br` is one switch case + one test.
- ~10 LOC source change, mirroring the v0.3 dep-bump pattern documented in Gate 10's "patch-release dep-bump pattern" finding.

### Tooling expectations

Phase 12 reuses all Phase 7–11 tooling without extension:

- `bare-swift new` is not needed (no new packages — only v0.2 / v0.4 bumps of existing repos).
- `bare-swift gen-site --umbrella .` after each version bump (the umbrella shows the new version automatically).
- Pre-emptive Pages tag-policy is already set on swift-brotli and swift-content-encoding from their v0.1 / v0.3 shipments.
- Strict-concurrency in CI stays mandatory.
- `docc-target: Brotli` is already set in swift-brotli's `ci.yml` from v0.1.

**New convention from Gate 11:** `bare-swift new` should set `docc-target: <Module>` by default in scaffolded `.github/workflows/ci.yml`. Optional pre-Phase-12 task; not blocking.

### Versioning

Two existing packages get version bumps:

| Package | New | Future |
|---|---|---|
| swift-brotli | v0.2.0 (decoder unchanged, encoder added) | v0.3 = streaming if adopter signal surfaces |
| swift-content-encoding | v0.4.0 (br encode branch added) | v0.5 if a new coding lands |

Phase 12 introduces **0 new packages** — the ecosystem stays at **43 packages**. v0.2 / v0.4 bumps only.

### Anti-goals

- **No new packages.** Phase 12 is a version-bump-only phase. New packages wait for Phase 13+.
- **No streaming encoder.** Adds significant API-surface and incremental-state complexity. v0.3 if demanded.
- **No custom dictionary support.** RFC 7932 static dictionary only.
- **No retroactive bug-fix sweep across prior phases.**

## Alternatives considered

### Anchor Phase 12 on internationalization (swift-idna, swift-publicsuffix)

**Pros:** closes swift-uri's IDNA deferral; data-heavy precedent already set by swift-brotli's 122 KiB dictionary.

**Cons:** IDNA is famously fiddly (UTS #46 + UTS #15 + Punycode); PSL is ~150 KiB data. Two packages of comparable density to swift-brotli v0.1. An entire phase on its own. Better suited as Phase 13 anchor once brotli v0.2 closes the compression tier.

**Verdict:** rejected as Phase 12. Phase 13 anchor.

### Anchor Phase 12 on OTLP cross-signal v0.2

**Pros:** closes Phase 2/3/4 single-signal deferral.

**Cons:** narrower audience than the compression tier completion. Three packages of medium complexity, each requiring protobuf encoder updates.

**Verdict:** rejected as Phase 12. Phase 14+ candidate.

### Anchor Phase 12 on swift-jwt-verify v0.2 (signing + RS256)

**Pros:** complements the just-shipped v0.1; closes the verify-only design boundary.

**Cons:** swift-jwt-verify v0.1 just shipped (Phase 11 Tranche 11C). Better to let v0.1 settle and gather adopter signal before expanding scope to signing. Phase 12 single-anchor on signing risks scope creep with no concrete demand yet.

**Verdict:** rejected as Phase 12. v0.2 minor release if adopter demand surfaces; otherwise wait.

### Anchor Phase 12 on OAuth 2.0 client primitives

**Pros:** sits naturally atop swift-bearer; closes the auth composition story.

**Cons:** no concrete adopter demand yet. OAuth 2.0 has many flows (auth code, client credentials, refresh, PKCE, device code) — picking the right v0.1 subset needs adopter input. Premature.

**Verdict:** rejected as Phase 12. Defer until concrete demand.

### Skip ahead, do crypto-adjacent (swift-blake3 / swift-siphash / swift-hkdf)

**Pros:** would complement swift-jwt-verify if signing eventually lands.

**Cons:** swift-crypto remains entrenched. Ninth rejection candidate across gates. Differentiation hasn't strengthened.

**Verdict:** rejected.

## Resolution

**Accepted 2026-05-12.** Phase 12 plans may begin authoring. First package: swift-brotli v0.2.0.

Phase 12 closes the compression tier's last asymmetry by adding the brotli encoder, completing the symmetric request/response compression story across all five web codings. Phase 13 anchor decision (likely internationalization) will land at Gate 12.

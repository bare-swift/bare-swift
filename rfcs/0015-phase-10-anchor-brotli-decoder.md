# RFC-0015 — Phase 10 anchor: swift-brotli (decoder)

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-12 |
| Resolution | Accepted 2026-05-12 — Phase 10 plans may begin authoring. First package: swift-brotli (decoder). |

## Summary

Anchor Phase 10 on **swift-brotli (decoder)** — a single-package, single-anchor phase. swift-brotli is the longest-deferred candidate in the ecosystem (declined as a stretch in Phases 7D, 8D, and 9D); shipping it closes a three-phase deferral and fills the most-visible compression gap in the bare-swift HTTP story. Phase 10 may include an optional 10B stretch (swift-content-encoding v0.3 wiring `br` into the multiplexer, or a first auth-primitives package), but the headline is brotli alone.

The decoder-first staging pattern committed in RFC-0012 (v0.1 decode, v0.2 encode) continues: Phase 10 brings swift-brotli to v0.1 (decode); the encoder waits for a future minor release.

## Problem

[Gate 9 retrospective](../docs/gates/2026-05-12-gate-9-retrospective.md) item 5 requires "Phase 10 anchor decision recorded as an RFC" before Phase 10 plans begin. The retrospective surveyed five candidate waves (swift-brotli decoder, auth primitives, OTLP v0.2 cross-signal, internationalization, crypto-adjacent); this RFC formalizes the choice.

After Phase 9, the bare-swift compression tier provides symmetric Foundation-free codecs for DEFLATE / gzip / zlib + an HTTP `Content-Encoding` multiplexer. The most-visible remaining gap is **Brotli** ([RFC 7932](https://www.rfc-editor.org/rfc/rfc7932.html)) — the default compression for HTTPS responses in Chrome, Edge, and Safari since 2017. Any modern HTTP client without Brotli support either falls back to gzip (losing 15–25% in payload size) or pulls in a non-Foundation-free library. The Swift-server ecosystem currently has no standalone Foundation-free Brotli decoder.

swift-brotli has been on the candidate list for three consecutive phases:

- **Phase 7 (compression tier)** — RFC-0012 listed it as a 7D stretch alternative; declined in favor of swift-content-encoding (multiplexer was a better fit at the time).
- **Phase 8 (HTTP request/response primitives)** — RFC-0013 listed it as an 8D stretch alternative; declined in favor of swift-link-header (smaller, faster to ship).
- **Phase 9 (compression-encoder sweep)** — RFC-0014 listed it as a 9D stretch alternative; declined in favor of swift-content-encoding v0.2 (closed the encoder symmetry).

Continuing to defer it indefinitely risks the "swift-brotli will never ship" anti-pattern. Phase 10 commits to closing it.

## Proposal

### Anchor: swift-brotli (decoder)

Phase 10 primary work is a single substantial package:

- **swift-brotli v0.1** — RFC 7932 decoder. Stream parser, ring-buffer-backed window (LZ77-style), prefix-code decoder (Brotli's Huffman-like complex prefix-codes with context-aware literals/insert-and-copy/distance alphabets), static-dictionary integration (RFC 7932 § 8 — the 120 KiB shared dictionary that Brotli ships with), context-modeling for literals.

Estimated source size: **~800–1000 LOC** depending on how aggressively the static dictionary is embedded (the 120 KiB binary blob ships as a Swift array literal or via a `.copy` resource — to be decided in the plan). Comparable in density to swift-deflate v0.1 INFLATE (Phase 7A) but with one substantial additional complexity: the static dictionary.

### Tranches

| Tranche | Packages | Estimated calendar |
|---|---|---|
| 10A | swift-brotli (decoder) — the only baseline | ~3–4 days at observed pace |
| 10B *(optional stretch)* | swift-content-encoding v0.3 (adds `br` branch to decode dispatch) OR swift-jwt-verify v0.1 OR swift-basic-auth v0.1 | ~0.5 day (content-encoding) or ~2 days (auth) |

Total Phase 10 budget: ~1 week wall-clock at observed pace. Package count (1 baseline + 1 stretch) is the smallest since Phase 1, but **LOC dominates** — the brotli decoder is the largest single deliverable since swift-deflate v0.1 INFLATE (Phase 7A).

### Why this anchor

1. **Closes a three-phase deferral.** swift-brotli has been declined three times running. Each defer was justified at the time; collectively they signal that no smaller-scoped wave will subsume it. The right move is to anchor on it directly.
2. **Adoption fit is highest.** Brotli is the default compression for HTTPS responses in every major browser since 2017. Any HTTP client must handle it; bare-swift has no answer. The gap is real and well-bounded.
3. **Single-anchor phase fits the package's substance.** Brotli decoder is large enough (~800 LOC) to anchor on its own — comparable to swift-deflate v0.1 INFLATE (Phase 7A) but with an additional complexity (static dictionary). Bundling three thin wrappers would dilute the focus.
4. **Decoder-first staging pattern continues.** Phase 7 established it (v0.1 decoder, v0.2 encoder). Phase 9 closed the encoder side for deflate/gzip/zlib. Phase 10 brings brotli into the same staging: v0.1 decoder, future v0.2 encoder.
5. **Builds on existing foundations.** swift-bytes (Phase 2) handles input/output buffers; swift-content-encoding (Phase 7+9) provides the `Content-Encoding` dispatch slot for `br`; Phase 9's encoder-staging discipline applies directly when the v0.2 encoder eventually ships.
6. **Sets up Phase 11+ options.** Once brotli ships:
   - **Auth primitives** become the natural Phase 11 anchor (HTTP server primitive stack completion).
   - **swift-brotli v0.2 encoder** lands as a focused minor release in any future phase.
   - **OTLP cross-signal** stays available as Phase 12+.

### Tranche 10A scope sketch

`swift-brotli` v0.1 covers RFC 7932:

- `Brotli.decode(_:) throws(BrotliError) -> Bytes` — single-shot decompression. Input is a raw Brotli bit stream; output is `Bytes`.
- `BrotliError` typed-throws enum covering truncation, invalid prefix codes, distance-out-of-range, invalid block-type sequences, dictionary lookup out of range, output-too-large guard.
- Internal types: a `BrotliBitReader` (LSB-first like DEFLATE's, but with the Brotli twist that the first bit is MSB of the first byte — careful spec reading required), a prefix-code decoder, a ring-buffer-backed window, the static dictionary loader, context-modeling tables.
- Static dictionary: 120 KiB of literal data per RFC 7932 § 8 + Appendix A. Two viable encodings:
  - Embed as a Swift array literal in source (~30K lines of byte literals; one-time compile cost).
  - Ship as a `Resources/brotli-dictionary.bin` and load via `Bundle.module` — **but this requires Foundation**, so it's incompatible with the Foundation-free invariant. Rejected.
  - Embed via a generated `.swift` source file (a script reads the binary at build time and writes a Swift array literal). **Preferred**: keeps the binary out of git but the runtime out of Foundation.
- Output cap (default 64 MiB) to prevent decompression bombs; configurable per-call. Mirrors swift-deflate v0.1's 32 MiB cap.

Round-trip property: for every `bytes` produced by a reference encoder (e.g. `python -m brotli` or the Google `bro` command-line tool, run **at ship time in a separate consumer**, not inside the test target), `Brotli.decode(bytes)` returns the original input. Inline test vectors are hand-curated from the RFC 7932 reference suite.

Out of scope for v0.1:

- **Encoder.** Per RFC-0012's staging pattern, the encoder ships in a future v0.2 (separate phase; not committed here). Brotli encoding is famously a multi-mode-context-modeling problem — substantially larger than the decoder.
- **Streaming decoding.** v0.1 takes a single full `Bytes` input.
- **Custom dictionaries.** RFC 7932 supports user-supplied dictionaries via the bit stream's reserved-bits path; v0.1 only handles the standard static dictionary.
- **`compress` (LZW), `zstd`, anything else not Brotli.** Out of scope; tracked as Phase 12+ candidates.

### Tranche 10B scope sketch (optional stretch)

The 10B stretch slot is optional and small. Three picks, in priority order:

**Option A: swift-content-encoding v0.3** (recommended default). Adds `br` and `x-br` aliases to the existing decode multiplexer's switch statement. Adds the same to the v0.2 encode mirror with a placeholder that throws `.unsupportedEncoding("br")` for encode (until swift-brotli v0.2 ships). ~30 LOC + tests + CHANGELOG. **~0.5 day.**

**Option B: swift-jwt-verify v0.1.** Verify-only (no signing) of JWS-compact JWTs against a pinned public key. ES256 + RS256 + HS256 minimum. Pulls in swift-crypto for the actual signature verification (acceptable — swift-crypto is explicitly NOT a bare-swift target; we wrap it). ~400 LOC. **~2 days.**

**Option C: swift-basic-auth v0.1.** RFC 7617 parser/serializer for `Authorization: Basic ...` headers. Tiny; ~50 LOC. **~0.5 day.** Less impactful than Option A since 10A already establishes the value of the phase.

Decision deferred to the executor at 10B start, in consultation with then-current adopter signal. **Default is Option A** if no signal exists — it composes naturally with the 10A deliverable.

### Out of scope for Phase 10

- **swift-brotli encoder.** Beyond decoder. Substantial multi-mode encoding problem; separate phase if/when prioritized.
- **OTLP v0.2 cross-signal sweep.** Still deferred. Phase 12+ candidate.
- **Crypto-adjacent packages.** Seventh-rejection candidate; revisit Phase 12+.
- **Internationalization (swift-idna, swift-publicsuffix).** Still niche; Phase 12+.
- **Streaming compression** across the tier. v0.3 anchor candidate, not Phase 10.

### Tooling expectations

Phase 10 reuses all Phase 7+8+9 tooling without extension:

- `bare-swift new` for the new swift-brotli package scaffold.
- `bare-swift gen-site --umbrella .` after the v0.1 release.
- Pre-emptive Pages tag-policy stays canonical (now seven-for-seven green-on-first-tag if Phase 10 holds the pattern).
- Strict-concurrency in CI stays mandatory.
- `docc-target` opt-in from scaffold time.
- Inline test vectors stay the rule; **cross-decoder validation against `bro` or `python -m brotli` runs at ship time in a separate /tmp/ consumer script**, not inside the test target — per Foundation-free-test-target invariant codified in Gate 9.
- v0.1 stability suite pattern doesn't apply (this is a v0.1 release, no prior version to stabilize against).

### Versioning

| Package | v0.1 (this phase) | Future |
|---|---|---|
| swift-brotli | decoder | v0.2 = encoder (separate phase) |
| swift-content-encoding *(if 10B picks A)* | v0.2 decoder + encoder mirror | v0.3 = add `br` branch (this phase's 10B) |

Phase 10 introduces one new package (swift-brotli) and possibly one minor release (swift-content-encoding v0.3).

### Anti-goals

- **No retroactive bug-fix sweep across Phase 9 packages.** Phase 10 is brotli-focused; deflate/gzip/zlib/content-encoding bug fixes land as their own v0.2.x patches independent of this phase.
- **No new tier on the umbrella.** Brotli joins the existing "compression" tier (along with deflate/gzip/zlib/content-encoding). The tier-grouping logic absorbs it without changes.
- **No swift-crypto dep in the baseline.** swift-brotli is pure-algorithmic and Foundation-free; no crypto needed. (10B Option B would introduce swift-crypto as a dep for swift-jwt-verify, which is acceptable since swift-crypto is explicitly a wrap-not-reimplement target per the README.)

## Alternatives considered

### Anchor Phase 10 on auth primitives (JWT verify / Basic / Bearer)

**Pros:** natural Phase-8 follow-on; closes the HTTP-server primitive stack. Three coherent small packages.

**Cons:** extends the brotli deferral one more phase, making four-in-a-row. Auth is also less audience-overlapping than brotli — every HTTP client needs brotli; only some need JWT.

**Verdict:** rejected. Strong Phase 11 candidate.

### Anchor Phase 10 on OTLP v0.2 cross-signal sweep

**Pros:** closes Phase 2/3/4's single-signal deferral.

**Cons:** extends the brotli deferral. Audience narrower (OTLP-only adopters) than brotli's universal HTTP-client audience.

**Verdict:** rejected. Phase 12+ candidate.

### Anchor Phase 10 on internationalization (swift-idna, swift-publicsuffix)

**Pros:** closes swift-uri's IDNA deferral; underrepresented in the ecosystem.

**Cons:** niche compared to brotli; IDNA is famously fiddly (UTS #46 + #15 + Punycode) and would consume a Phase's worth of work for narrow audience.

**Verdict:** rejected. Phase 12+ side-deliverable candidate.

### Skip Phase 10 entirely; jump to a four-tranche "Phase 10 = JWT, Basic, Bearer, content-encoding-v0.3" wave

**Pros:** more packages, fits the canonical four-tranche shape.

**Cons:** extends the brotli deferral indefinitely. The four-tranche shape is convention, not requirement; single-anchor phases are valid when the headline package is substantial.

**Verdict:** rejected. The brotli deferral cost compounds; single-anchor is the right structural choice for this gap.

### Don't anchor at all; let swift-brotli happen "on-demand"

**Pros:** lower coordination overhead.

**Cons:** swift-brotli has been "on-demand" for three phases and hasn't shipped. The on-demand experiment failed.

**Verdict:** rejected.

## Resolution

**Accepted 2026-05-12.** Phase 10 plans may begin authoring. First package: swift-brotli (decoder).

Phase 10 closes the longest-standing single-package deferral in the ecosystem and fills the most-visible Brotli-shaped gap in the bare-swift HTTP story. Phase 11 anchor decision (likely auth primitives) will land at Gate 10.

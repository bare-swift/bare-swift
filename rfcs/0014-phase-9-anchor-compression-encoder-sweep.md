# RFC-0014 — Phase 9 anchor: Compression-encoder v0.2 sweep

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-11 |
| Resolution | Accepted 2026-05-11 — Phase 9 plans may begin authoring. First package: swift-deflate v0.2 (RFC 1951 encoder). |

## Summary

Anchor Phase 9 on the **compression-encoder v0.2 sweep**: ship the encoder half of every package in the compression tier introduced by RFC-0012 (Phase 7). swift-deflate v0.2 (RFC 1951 LZ77 + canonical-Huffman encoder) is the headline; swift-gzip v0.2 and swift-zlib v0.2 follow as thin wrappers. swift-content-encoding v0.2 wires the encoder half into the multiplexer as the optional 9D stretch — alternative 9D stretches are swift-brotli (decoder) or swift-otlp-traces v0.2.

The sweep closes the explicit deferral committed by RFC-0012 ("v0.1 = decoder, v0.2 = encoder; encoder ships once the decoder is stable"). Phase 7 decoders have now been stable for ~1 day with 55 tests and zero post-release issues; the encoder commitment is due.

## Problem

[Gate 8 retrospective](../docs/gates/2026-05-11-gate-8-retrospective.md) item 5 requires "Phase 9 anchor decision recorded as an RFC" before Phase 9 plans begin. The retrospective surveyed six candidate waves (compression encoders v0.2, crypto-adjacent, internationalization, OTLP variants, auth primitives, HTTP body v0.2); this RFC formalizes the choice.

[RFC-0012 § Out of scope](./0012-phase-7-anchor-http-body-codecs.md) committed:
> "v0.1 of every compression package ships the **decoder** only. The encoder lands in a minor v0.2 once the decoder is stable across real-world traffic."

After Phase 7 (decoders) and Phase 8 (HTTP request/response primitives), the encoder commitment is the longest-standing deferred deliverable. Continuing to defer it risks the "v0.2 that never ships" anti-pattern. Phase 9 commits to closing it.

The bare-swift ecosystem at 38 packages has every primitive a Foundation-free HTTP server needs for request *decoding* — URIs, MIME types, cookies, compression decompression, multipart, ranges, conditional caching, link headers. The encoder side of compression is the last missing piece for symmetric request/response handling. A server that can decompress an `Accept-Encoding: gzip` request body but cannot compress its response is half a story.

## Proposal

### Anchor: compression-encoder v0.2 sweep

Phase 9 primary work targets the four compression-tier packages:

- **swift-deflate v0.2** — RFC 1951 encoder. LZ77 match-finding (hash-chain or binary-tree), canonical-Huffman tree construction, dynamic vs fixed vs stored block-type selection, bit-level output. This is the headline package — substantial complexity (~1500 LOC), comparable in density to the v0.1 decoder.
- **swift-gzip v0.2** — RFC 1952 encoder. Wraps swift-deflate v0.2's encoder with the RFC 1952 member header (FLG / MTIME / XFL / OS), optional FNAME / FCOMMENT fields, trailing CRC-32 + ISIZE. Thin wrapper (~150 LOC).
- **swift-zlib v0.2** — RFC 1950 encoder. Wraps swift-deflate v0.2's encoder with the RFC 1950 header (CMF + FLG) and trailing ADLER32. Thin wrapper (~80 LOC; ADLER32 is already inlined from v0.1).
- **swift-content-encoding v0.2** *(stretch)* — wires the encoder half into the multiplexer. Adds `Encoder` to the existing `Decoder` enum, threads it through the same multi-coding logic. Thin wrapper (~100 LOC).

Alternative 9D stretches:

- **swift-brotli** (RFC 7932 decoder) — the longest-deferred candidate from Phases 7 & 8. Decoder is moderate complexity (~800 LOC; dictionary-based + Huffman-like with several encoding modes).
- **swift-otlp-traces v0.2** — extends swift-otlp-exporter (Phase 2B) with the traces signal. Builds on swift-tracing-otlp (Phase 3A) for the protobuf shape; mostly multiplexing existing pieces.

### Tranches

| Tranche | Packages | Estimated calendar |
|---|---|---|
| 9A | swift-deflate v0.2 — RFC 1951 encoder (LZ77 + canonical Huffman + dynamic blocks) — the headline | ~2–3 days at observed pace |
| 9B | swift-gzip v0.2 — RFC 1952 encoder wrapper | ~0.5 day |
| 9C | swift-zlib v0.2 — RFC 1950 encoder wrapper | ~0.5 day |
| 9D *(stretch)* | swift-content-encoding v0.2 OR swift-brotli (decoder) OR swift-otlp-traces v0.2 | ~1–2 days |

Total Phase 9 budget: ~1 week calendar at observed pace. Package count (3 baseline + 1 stretch) is the smallest since Phase 3, but **LOC dominates** because the DEFLATE encoder is the largest single deliverable since swift-uri (Phase 4) and swift-deflate v0.1 INFLATE (Phase 7).

### Why this anchor

1. **Honors a committed deferral.** RFC-0012 explicitly committed the encoder half. Gate 7 deferred *when* to Gate 8. Gate 8 (this RFC's parent retrospective) is now. Picking anything else stretches the deferral to Phase 10+ and risks "v0.2 that never ships."
2. **Anchor-worthy on its own.** DEFLATE encoder is large enough to anchor independently. Comparable to swift-uri's WHATWG state machine (Phase 4A) — algorithmic, well-specified, no upstream port to crib from cleanly.
3. **Builds on Phase 7 foundations.** swift-deflate's BitReader / canonical-Huffman tables from v0.1 were designed to be encoder-reusable. The encoder adds a parallel `BitWriter` plus LZ77 match-finding, then composes against the existing Huffman code paths.
4. **Audience continuity is total.** Same HTTP server/client implementers Phase 7 + 8 serve. Symmetric request/response handling completes the story.
5. **Risk profile is well-understood.** DEFLATE encoding correctness is unambiguous: encode → decode (via swift-deflate v0.1) → byte-equal. The *tuning* problem (zopfli vs gzip -9 vs gzip -6 produce different sizes for byte-equal inputs) is **out of scope for v0.2** — v0.2 commits to correctness only; size/speed tuning lands as v0.2.x patch releases.
6. **Sets up Phase 10+ options.**
   - **swift-brotli** becomes a natural Phase 10 anchor (the most-deferred candidate, now ready when DEFLATE encoders are done).
   - **Auth primitives** (Basic Auth, Bearer, JWT verify) become approachable.
   - **OTLP v0.2 cross-signal** (traces, logs, json mode) closes the observability deferral.

### Tranche 9A scope sketch

`swift-deflate` v0.2 adds:

- `Deflate.Encoder` value type — encodes `Bytes` input into a DEFLATE bit stream.
- `Deflate.Encoder.Level` enum — quality levels (`.none` = stored blocks only, `.fast` = fixed Huffman, `.default` = dynamic Huffman with fast match-finding, `.best` = dynamic Huffman with thorough match-finding). Default `.default`.
- LZ77 match-finding via hash-chain with configurable max-chain-length per level.
- Canonical-Huffman tree construction (Huffman length-limited per RFC 1951 § 3.2.7).
- Dynamic vs fixed vs stored block-type selection per block (choose smallest output).
- `Deflate.encode(_:level:)` convenience function for one-shot use.
- `DeflateError` extended (if needed) with encoder-specific cases (e.g. `inputTooLarge`).

Round-trip property: for every `bytes`, `Deflate.decode(Deflate.encode(bytes)) == bytes`. Validated by `swift test` and a consumer round-trip against `python -m zlib.decompress`.

Out of scope for v0.2:

- Streaming encoding for arbitrarily-large inputs. v0.2 takes a single full `Bytes` input.
- Zopfli-style size optimization (multiple-pass encoding). Future v0.2.x patch.
- Preset dictionary encoding. Future v0.3 if requested.
- Custom Huffman dictionaries beyond the RFC 1951 canonical pair.

### Tranche 9B scope sketch

`swift-gzip` v0.2 adds:

- `Gzip.Encoder` value type — wraps `Deflate.Encoder`, adds RFC 1952 member header + trailer.
- Convenience `Gzip.encode(_:filename:modificationTime:level:)`.
- CRC-32/ISO-HDLC of the *uncompressed* input (computed via swift-crc).
- ISIZE = uncompressed length modulo 2^32.
- Optional FNAME / FCOMMENT fields (null-terminated ASCII; non-ASCII filenames omitted with a warning in DocC, not in throws — RFC 1952 § 2.3.1.4 says original-name field is OS-specific).

Out of scope for v0.2:

- Multi-member gzip streams (concatenated `.gz` files). v0.1 decoder already rejects multi-member; v0.2 encoder produces single-member only.
- FHCRC (header CRC). Optional in RFC 1952; v0.2 omits.

### Tranche 9C scope sketch

`swift-zlib` v0.2 adds:

- `Zlib.Encoder` value type — wraps `Deflate.Encoder`, adds RFC 1950 header + trailer.
- Convenience `Zlib.encode(_:level:)`.
- ADLER32 of the *uncompressed* input (already inlined in v0.1).
- CMF / FLG / FLEVEL bits per RFC 1950 § 2.2.

Out of scope for v0.2:

- Preset dictionary (FDICT bit set). v0.2 always produces FDICT=0.

### Tranche 9D scope sketch (stretch options)

**Option A: swift-content-encoding v0.2** — adds the encoder side to the existing multiplexer. Same approach as the v0.1 decoder: parse `Content-Encoding` header → pick encoder per coding → run pipeline left-to-right. Closes the symmetry. ~100 LOC.

**Option B: swift-brotli** — RFC 7932 decoder. The most-deferred candidate from Phases 7 & 8. Decoder only (matching the v0.1 staging pattern); encoder would be a future v0.2.

**Option C: swift-otlp-traces v0.2** — extends swift-otlp-exporter with the traces signal. Lower priority than closing compression but available as an alternative if compression encoders ship early.

Decision deferred to the executor at 9D start, in consultation with then-current adopter signal.

### Out of scope for Phase 9

- **swift-brotli encoder** (beyond decoder if 9D picks Option B). Brotli encoding is famously complex (a multi-mode encoder with context modeling); separate phase if/when prioritized.
- **OTLP v0.2 cross-signal sweep beyond traces.** Logs and json mode wait for a dedicated phase.
- **Crypto-adjacent packages.** Sixth-rejection candidate; revisit Phase 11+.
- **Internationalization (swift-idna, swift-publicsuffix).** Still niche; Phase 11+.
- **HTTP body v0.2 deferrals** (multipart streaming, range × conditional If-Range integration). Land as v0.2.x patch releases when consumers ask.

### Tooling expectations

The encoder sweep should reuse all Phase 7+8 tooling without extension:

- `bare-swift new` for new test consumers (no new packages — these are minor releases of existing packages).
- `bare-swift gen-site --umbrella .` after each version bump.
- The pre-emptive Pages tag-policy pattern stays canonical; the four compression packages already have Pages configured.
- Strict-concurrency in CI stays mandatory.
- `docc-target` already opt-in across all four packages from v0.1.
- Inline test vectors stay the rule; encoder validation uses **round-trip-then-cross-decoder** (encode with us → decode with `python -m zlib.decompress` or `gzip -d`) for high-confidence correctness.

### Versioning

Each package gets a v0.2.0 tag:

| Package | v0.1 | v0.2 (this phase) |
|---|---|---|
| swift-deflate | decoder | + encoder |
| swift-gzip | decoder | + encoder |
| swift-zlib | decoder | + encoder |
| swift-content-encoding *(if 9D picks A)* | decoder pipeline | + encoder pipeline |

The v0.1 API surface stays intact in v0.2 (additive only). Consumers depending on `from: "0.1.0"` continue to work; consumers needing the encoder pin `from: "0.2.0"`.

### Anti-goals

- **No retroactive bug-fix sweep.** v0.2 is encoder additions, not v0.1 fixes. Any decoder bug fix gets its own v0.1.x patch release, independent of Phase 9.
- **No package renames / module renames.** The `HTTPRange`-style stdlib-collision fix from Phase 8 doesn't apply here — `Deflate` / `Gzip` / `Zlib` / `ContentEncoding` don't shadow stdlib types.
- **No new dependencies.** swift-deflate already depends on swift-bytes; swift-gzip on swift-crc + swift-deflate; swift-zlib on swift-deflate; swift-content-encoding on swift-deflate + swift-gzip + swift-zlib. Phase 9 should add zero new edges to the dependency graph.

## Alternatives considered

### Anchor Phase 9 on swift-brotli (decoder)

**Pros:** longest-deferred candidate; meaningful adopter signal.

**Cons:** extends the encoder-deferral one more phase; swift-brotli ships as an island while the encoder commitment continues to age.

**Verdict:** rejected. swift-brotli is a strong Phase 10 candidate (after the encoder sweep clears).

### Anchor Phase 9 on auth primitives (JWT verify, Basic, Bearer)

**Pros:** natural follow-on to Phase 8's HTTP primitives; complete the request-handling story.

**Cons:** same as above — extends the encoder deferral. Also more design-space-uncertain than the encoder sweep (multiple JWT libraries differ on which algs to support, key formats, etc.).

**Verdict:** rejected as Phase 9. Strong Phase 10+ candidate.

### Anchor Phase 9 on OTLP v0.2 cross-signal sweep

**Pros:** closes another long-standing deferral (Phase 2/3 single-signal scoping).

**Cons:** extends the encoder deferral. Also: less audience overlap with Phases 7+8 than the encoder sweep.

**Verdict:** rejected as Phase 9. Phase 11+ candidate after encoders + brotli + auth.

### Don't anchor at all; let v0.2 releases happen on-demand

**Pros:** lower coordination overhead; aligns with semver minor-release pattern.

**Cons:** "on-demand" has not produced a single v0.2 across four packages in two phases. RFC-0012's commitment ages without progress. Anchoring forces the work.

**Verdict:** rejected. The deferral has been "on-demand" since RFC-0012; that experiment didn't produce results.

## Resolution

**Accepted 2026-05-11.** Phase 9 plans may begin authoring. First package: swift-deflate v0.2.

Phase 9 closes the longest-standing deferral in the ecosystem and completes the compression tier as a symmetric request/response codec library. Phase 10 anchor decision (likely swift-brotli or auth primitives) will land at Gate 9.

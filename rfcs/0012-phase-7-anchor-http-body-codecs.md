# RFC-0012 — Phase 7 anchor: HTTP body codecs / compression wave

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-10 |
| Resolution | Accepted 2026-05-10 — Phase 7 plans may begin authoring. First package: swift-deflate (INFLATE in v0.1; DEFLATE encoder deferred to v0.2). |

## Summary

Anchor Phase 7 on the **HTTP body codecs / compression wave**: ship swift-deflate (RFC 1951 INFLATE) as the headline package, then swift-gzip (RFC 1952 wrapper) and swift-zlib (RFC 1950 wrapper); swift-content-encoding (an HTTP `Content-Encoding` header multiplexer) or swift-brotli (RFC 7932) as the optional 7D stretch. v0.1 of each compression package focuses on **decompression**; the encoder side ships as a v0.2 minor release once the decoder is stable.

The wave reuses Phase 2 / Phase 4 / Phase 6 foundations (swift-bytes for byte streams, swift-mime for content-type negotiation if 7D ships swift-content-encoding), targets the largest unfilled gap in the ecosystem (Foundation-free DEFLATE/INFLATE genuinely doesn't exist in Swift today — `Compression.framework` is Foundation-bound and Apple-platform-only), and unblocks downstream waves (OTLP/json with gzip-encoded payloads, content-negotiating HTTP frameworks, pre-compressed static asset servers).

## Problem

[Gate 6 retrospective](../docs/gates/2026-05-10-gate-6-retrospective.md) item 5 requires "Phase 7 anchor decision recorded as an RFC" before Phase 7 plans begin. The retrospective surveyed five candidate waves (HTTP body codecs, crypto-adjacent, internationalization, OTLP/json, HTTP-types-extras); this RFC formalizes the choice.

After 30 packages across six phases, the ecosystem has the foundation tier (bytes, varint, base64, crc, hex, uuid, xxhash, time), the format tier (jsonpointer, dotenv, mime, cookie, uri, msgpack, cbor, toml, json, jsonpath, jsonpatch, jsonschema-lite — 12 packages), the observability tier (statsd, prometheus, hdrhistogram, ddsketch, otlp-exporter, prometheus-metrics, tracing-otlp, distributed-tracing-bridge, log-otlp). Conspicuously absent: **byte-stream codecs**. Every HTTP server/client decompresses incoming `Content-Encoding: gzip` request bodies and encodes outgoing responses; the Swift ecosystem's options are `Compression.framework` (Apple-platform-only, Foundation-bound) or `zlib.h` C bindings.

This RFC commits to closing that gap.

## Proposal

### Anchor: HTTP body codecs / compression wave

Phase 7 primary work targets the byte-stream codecs every HTTP server / client needs:

- **swift-deflate** — RFC 1951 INFLATE (decompression). DEFLATE encoder deferred to v0.2 per the staging pattern (decompression first, compression second). This is the headline package — RFC 1951 is moderately complex (Huffman tables, sliding window) but the spec is well-bounded.
- **swift-gzip** — RFC 1952 wrapper around DEFLATE (gzip framing: header, CRC32 trailer, ISIZE). Calls swift-deflate under the hood.
- **swift-zlib** — RFC 1950 wrapper (zlib framing: 2-byte header, ADLER32 trailer). Calls swift-deflate under the hood. zlib is what HTTP `Content-Encoding: deflate` actually means in practice (per RFC 7230 § 4.2.2 — the "deflate" name is historically misleading).
- **swift-content-encoding** *(stretch)* — HTTP `Content-Encoding` header multiplexer. Composes swift-mime (header parsing) + swift-gzip / swift-zlib / passthrough into a single `decode(_ bytes: Bytes, contentEncoding: String) throws -> Bytes` interface. Server-stack convenience.

Alternative 7D: **swift-brotli** (RFC 7932) — a genuinely different algorithm, not a wrapper. Substantially more complex than DEFLATE (Huffman + dictionary preprocessing + context modeling). Skip in v0.1 unless an early adopter requests it; defer to Phase 8+ if so.

### Tranches

| Tranche | Packages | Estimated calendar |
|---|---|---|
| 7A | swift-deflate (RFC 1951 INFLATE) — the headline | ~1–2 days at observed pace |
| 7B | swift-gzip (RFC 1952 wrapper) | ~1 day |
| 7C | swift-zlib (RFC 1950 wrapper) | ~0.5 day |
| 7D *(stretch)* | swift-content-encoding (HTTP header multiplexer) OR swift-brotli (RFC 7932) | ~1–2 days |

### The "INFLATE-only in v0.1" staging decision

DEFLATE compression is genuinely harder than INFLATE decompression. The compressor must run an LZ77 sliding-window search for back-references, build optimal-or-near-optimal Huffman tables for both literals/lengths and distances, and balance compression ratio against output buffer flushes. The decompressor reads pre-built Huffman tables from the input and applies them — substantial but bounded.

For HTTP server / client use, decompression is the dominant case:
- Servers receiving compressed request bodies → decompress.
- Clients receiving compressed responses → decompress.
- Servers compressing responses → encode (less common; many frameworks pre-compress static assets and skip on-the-fly compression).
- Clients compressing request bodies → encode (rare; most APIs accept uncompressed JSON).

Shipping decompression first delivers ~70% of the value at ~30% of the effort. The v0.2 sweep adds DEFLATE encoding once the decoder is stable and tested.

This staging pattern mirrors:
- **swift-uri v0.1**: ASCII-fast-path IDNA, v0.2 full UTS46.
- **swift-cbor v0.1**: float16 decode (lossless to float32), v0.2 may add encoding once a real consumer asks.

### Why this anchor

1. **Largest unfilled gap.** Every HTTP server / client deals with `Content-Encoding: gzip`. Foundation-free options are scarce; `Compression.framework` is Apple-platform-only and Foundation-bound, `zlib.h` requires a C-binding shim.
2. **Audience continuity is total.** Phase 4's networking primitives (uri, mime, cookie) and Phase 6's JSON tier consumers all eventually serve compressed HTTP bodies.
3. **Builds on Phase 2 + Phase 6 foundations.** swift-bytes is the natural input/output type. swift-mime parses `Content-Encoding` headers if 7D ships swift-content-encoding.
4. **Differentiation is real.** Foundation-free, no `Compression.framework` / `zlib.h` C-binding, pure Swift, value-type input and output. Genuinely missing.
5. **Risk profile is known.** RFC 1951 is well-bounded. INFLATE first means we ship a useful package even if DEFLATE encoding takes longer.
6. **Sets up Phase 8+.** Once compression exists: OTLP/json + gzip becomes a one-line Phase 8 wave; `Accept-Encoding` content negotiation in HTTP frameworks becomes trivial; static-file servers can serve pre-compressed assets.

### Tranche 7A scope sketch

`swift-deflate` covers RFC 1951 INFLATE in v0.1:

- `Deflate.inflate(_ compressed: Bytes) throws(DeflateError) -> Bytes` — single-shot decompression.
- Streaming API deferred to v0.2 alongside the encoder.
- Handles all three block types per RFC 1951:
  - Block type 00: stored (uncompressed).
  - Block type 01: fixed Huffman.
  - Block type 10: dynamic Huffman (the common case).
  - Block type 11: reserved → throws.
- Sliding-window back-references up to 32 KiB.
- Bit-level reading via a `BitReader` helper (LSB-first per RFC 1951).
- `DeflateError` typed-throws enum (truncated input, invalid block type, malformed Huffman table, distance out of range, etc.).

Out of scope for v0.1:
- DEFLATE compression (encoder). Defer to v0.2.
- Streaming partial decompression. v0.1 takes a single full payload.
- Block-by-block introspection (debugging APIs).
- Custom dictionary preset (RFC 1951 doesn't define one; zlib does — that lives in swift-zlib v0.2 if needed).

### Tranche 7B scope sketch

`swift-gzip` covers RFC 1952:

- `Gzip.decode(_ bytes: Bytes) throws(GzipError) -> Bytes` — strip gzip header, call deflate.inflate, validate CRC32 trailer.
- Multi-member streams (concatenated gzip files) — accept in v0.1; the spec allows them.
- Optional header fields (FNAME, FCOMMENT, FEXTRA, FHCRC) — skip past on read; encoder will support setting them in v0.2.
- `GzipError` typed-throws enum (bad magic, CRC32 mismatch, ISIZE mismatch, etc.).

Out of scope for v0.1:
- gzip compression (encoder). Defer to v0.2.
- Concatenated-stream incremental output. v0.1 returns the concatenation as one Bytes.

### Tranche 7C scope sketch

`swift-zlib` covers RFC 1950:

- `Zlib.decode(_ bytes: Bytes) throws(ZlibError) -> Bytes` — parse 2-byte zlib header, call deflate.inflate, validate ADLER32 trailer.
- HTTP `Content-Encoding: deflate` actually means zlib (per RFC 7230 § 4.2.2's note); v0.1 documents this prominently.
- `ZlibError` typed-throws enum.

Out of scope for v0.1:
- zlib compression (encoder). Defer to v0.2.
- Preset-dictionary support. The 4-byte DICTID in the header is parsed; using a preset dictionary on decode is v0.2.

### Out of scope for Phase 7

- Brotli (RFC 7932) — fundamentally different algorithm. May ship as 7D stretch if scoping is clean; otherwise defer to Phase 8+.
- zstd (RFC 8478) — different algorithm. Defer.
- LZ4 — defer.
- HTTP/2 HPACK header compression. Different problem domain.
- TLS-level compression (CRIME/BREACH-relevant). Out of audience.

## Alternatives considered

### Anchor on the crypto-adjacent wave (BLAKE3, SipHash, HKDF)

Rejected for the **fourth consecutive phase**. swift-crypto remains too entrenched. Four rejections is a strong signal this is not the right anchor for the bare-swift ecosystem at this scale.

### Anchor on the internationalization tier (swift-idna, swift-publicsuffix)

Rejected because:
- swift-uri's IDNA deferral is real but small. Non-ASCII hostnames are rare in server-stack code.
- swift-publicsuffix is data-heavy (~10K entries) and the use case (cookie domain validation) is niche.
- Compression has a strictly larger audience.

### Anchor on the OTLP/json variants wave

Rejected because:
- Audience overlap with compression is total. If compression ships first, OTLP/json + gzip becomes one Phase 8 wave.
- swift-otlp-json fills a real deferral but the value is incremental against existing protobuf encoders.
- Better timing: compression first, then OTLP/json wraps it.

### Anchor on HTTP-types-extras

Rejected because:
- Too narrow. Multipart, byte ranges, and header-validation extras are useful but don't form a coherent 3–4 tranche wave.
- Better as Phase 8+ side deliverables.

### Pick brotli as the headline instead of DEFLATE

Rejected because:
- DEFLATE/gzip is universal across HTTP. Brotli is widespread but not guaranteed (`br` requires the client to advertise support).
- Brotli is substantially more complex than DEFLATE (context modeling, dictionary preprocessing). Higher risk for the headline tranche.
- DEFLATE first means swift-content-encoding can multiplex {identity, gzip, deflate}; brotli can be added later.

## Unresolved questions

- **Should swift-deflate v0.1 expose the raw bit-level reader as a public API?** Useful for downstream compressors writing custom DEFLATE variants, but exposes implementation detail. Resolution: keep internal in v0.1; promote to public in v0.2 if a real consumer asks.
- **Streaming API shape for v0.2.** Most likely an `AsyncSequence<Bytes>` or callback-based incremental decoder. Resolution: design alongside the v0.2 encoder; defer concrete API decisions.
- **Pre-built Huffman tables for the fixed-block case (block type 01).** RFC 1951 specifies these; implementations either embed the constants or compute them once. Resolution: embed (constant arrays); v0.2 may switch if profiling shows the embed is dead weight in real workloads.
- **swift-content-encoding 7D stretch — does it depend on swift-mime?** swift-mime parses the `Content-Encoding` header value the same way it parses any other token list. Resolution: yes, it depends on swift-mime (also adds it as the first non-Phase-7 dep within Phase 7).

## Drawbacks

- **DEFLATE/INFLATE is genuinely complex.** ~600 LOC for inflate alone, plus careful bit-level testing. The largest single-package risk in the ecosystem so far.
- **No streaming API in v0.1.** Single-payload decompression has memory implications for large compressed payloads. Document the limitation and defer streaming to v0.2.
- **swift-content-encoding adds a dep (swift-mime) that 7A/7B/7C don't have.** Consider whether the multiplexer should live elsewhere (in swift-mime itself? in swift-http-types-extras?). Resolution: ship as its own package; swift-mime and swift-deflate are the natural deps; the package's job is bridging them.

## Migration impact

- **No breaking changes to existing packages.** All Phase 7 work is additive new packages.
- **swift-otlp-exporter / swift-tracing-otlp / swift-log-otlp eligible for `Content-Encoding: gzip`** once 7B ships. v0.3 of each may add a `gzipped()` wrapper that compresses the protobuf payload before HTTP POST. Documented as a future minor release, not committed in this RFC.
- **swift-uri / swift-mime / swift-cookie unaffected.** They produce header / wire fragments; compression is a body-level concern.

## Resolution

Accepted 2026-05-10. Phase 7 anchors on the **HTTP body codecs / compression wave** with three core tranches (7A swift-deflate, 7B swift-gzip, 7C swift-zlib) and one stretch slot (7D — swift-content-encoding or swift-brotli, decided at execution time). v0.1 of each ships **decompression**; the encoder side lands in v0.2.

Phase 7 plans may begin authoring; first package is **swift-deflate** (INFLATE).

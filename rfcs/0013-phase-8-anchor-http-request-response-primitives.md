# RFC-0013 — Phase 8 anchor: HTTP request/response primitives wave

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-10 |
| Resolution | Accepted 2026-05-10 — Phase 8 plans may begin authoring. First package: swift-multipart. |

## Summary

Anchor Phase 8 on the **HTTP request/response primitives wave**: ship swift-multipart (RFC 7578 multipart/form-data) as the headline package, then swift-range (RFC 7233 byte ranges) and swift-conditional (RFC 7232 ETag / If-* headers); swift-link-header (RFC 8288) or swift-brotli as the optional 8D stretch. The compression-encoder v0.2 sweep committed in RFC-0012 is deferred to Phase 9 as its own anchor — it's substantial enough (DEFLATE encoder ~1500 LOC) to warrant a dedicated wave.

The wave reuses the Phase 4 + Phase 7 foundations (swift-mime parses multipart's `Content-Type: ...; boundary=...`, swift-bytes handles raw bodies, swift-content-encoding handles compressed bodies), targets the next-most-requested gap in the swift-server adopter set, and unblocks Phase 9+ options (compression encoders, JWT verify, OTLP/json).

## Problem

[Gate 7 retrospective](../docs/gates/2026-05-10-gate-7-retrospective.md) item 5 requires "Phase 8 anchor decision recorded as an RFC" before Phase 8 plans begin. The retrospective surveyed six candidate waves (HTTP request/response primitives, crypto-adjacent, internationalization, OTLP/json variants, compression encoders v0.2 sweep, swift-brotli alone); this RFC formalizes the choice.

After 34 packages across seven phases, the ecosystem has the foundation tier (8 packages), the format tier (13 packages), the observability tier (9 packages), and the new compression tier (4 packages from Phase 7). What's still missing for a complete HTTP server / client stack: **request-body parsing** (multipart for file uploads), **byte-range requests** (partial downloads, resumable transfers), and **conditional caching** (ETag, If-Modified-Since). Every server framework — Vapor, Hummingbird, NIO-based handlers — re-implements these.

This RFC commits to closing that gap.

## Proposal

### Anchor: HTTP request/response primitives wave

Phase 8 primary work targets the HTTP body / header primitives every server / client framework needs:

- **swift-multipart** — RFC 7578 multipart/form-data parser + serializer. Boundary detection, per-part headers (Content-Type, Content-Disposition with name/filename), nested multipart support. This is the headline package — moderate complexity (binary boundary scanning with proper escape handling, RFC 5322 header grammar reuse).
- **swift-range** — RFC 7233 Range request handling. `Range: bytes=0-499, 500-999` and `Range: bytes=-500` (suffix) and `Range: bytes=9500-` (open-ended) parser; `Content-Range` response header serializer; helpers for slicing a `Bytes` against a parsed range.
- **swift-conditional** — RFC 7232 conditional request handling. `ETag` (strong + weak) parser/serializer, `If-Match` / `If-None-Match` / `If-Modified-Since` / `If-Unmodified-Since` evaluation against a resource's current ETag + last-modified.
- **swift-link-header** *(stretch)* — RFC 8288 `Link:` header parser/serializer (`<uri>; rel="..."; ...`). Smaller than the others; useful for pagination / preload hints.

Alternative 8D: **swift-brotli** (RFC 7932) — substantially more complex than DEFLATE; defer to a dedicated session.

### Tranches

| Tranche | Packages | Estimated calendar |
|---|---|---|
| 8A | swift-multipart (RFC 7578 parser + serializer) — the headline | ~1–2 days at observed pace |
| 8B | swift-range (RFC 7233 byte-range header parsing + slicing) | ~1 day |
| 8C | swift-conditional (RFC 7232 ETag + If-* headers) | ~1 day |
| 8D *(stretch)* | swift-link-header (RFC 8288) OR swift-brotli (RFC 7932) | ~1–2 days |

Total Phase 8 budget: 1–2 weeks calendar at observed pace; 4–6 months at original roadmap pace.

### Why this anchor

1. **Audience continuity is total.** Phase 4's networking primitives (uri, mime, cookie) and Phase 7's compression both serve HTTP server / client implementers. Phase 8 closes the next-most-requested gap in that audience: request-body parsing, byte ranges, conditional caching.
2. **Foundation-free differentiation is real.** The swift-server ecosystem has framework-bundled multipart parsers (Vapor, Hummingbird) but no standalone Foundation-free option. Range + conditional are similarly fragmented.
3. **Builds on Phase 4 + Phase 7 foundations.** swift-mime parses multipart's `Content-Type: ...; boundary=...` to extract the boundary; swift-bytes carries the raw body; swift-content-encoding handles compressed bodies before the multipart parser sees them.
4. **Risk profile is known.** Multipart's binary boundary detection is the most subtle piece (boundaries can appear inside content if not properly escaped — RFC 7578 says they shouldn't, but real-world payloads sometimes do); range and conditional are header-grammar parsers, smaller. All three packages are well-bounded.
5. **Sets up Phase 9+ options.** Once HTTP body primitives exist:
   - **Compression encoders v0.2 sweep** becomes the natural Phase 9 anchor (closes RFC-0012's deferred commitment).
   - **Auth primitives** (Basic Auth, Bearer, JWT verify) become approachable.
   - **OTLP/json mode** wraps the observability story.

### Tranche 8A scope sketch

`swift-multipart` covers RFC 7578:

- `MultipartParser` value type — parses a `Bytes` payload + a boundary string.
- `MultipartPart` value type — `headers: [String: String]`, `body: Bytes`.
- Helpers: `name`, `filename`, `contentType` from the Content-Disposition / Content-Type headers per part.
- `MultipartSerializer` — given `[MultipartPart]` + a boundary, produce a `Bytes` payload.
- `MultipartError` typed-throws enum.

Out of scope for v0.1:
- Streaming parsing of arbitrarily-large bodies. v0.1 takes a single full Bytes input.
- Multipart bodies *other* than form-data (multipart/mixed, multipart/byteranges, multipart/alternative). v0.1 narrows to form-data semantics; future v0.2 may generalize.
- File-upload-to-disk helpers. v0.1 returns Bytes; consumers stream to disk if needed.

### Tranche 8B scope sketch

`swift-range` covers RFC 7233:

- `RangeRequest` — parses `Range: bytes=0-499, 500-999`, `Range: bytes=-500` (suffix), `Range: bytes=9500-` (open).
- `ContentRange` — serializes / parses the response header (`Content-Range: bytes 0-499/1000`).
- `RangeRequest.satisfiable(against totalSize: Int) -> [Range<Int>]?` — returns the validated/clamped byte ranges or nil if unsatisfiable.
- `RangeError` typed-throws enum.

Out of scope for v0.1:
- Conditional ranges (`If-Range`). Defer to v0.2; intersects with swift-conditional.
- Multipart/byteranges response body construction. Out of scope for v0.1; consumers can compose with swift-multipart manually if needed.

### Tranche 8C scope sketch

`swift-conditional` covers RFC 7232:

- `ETag` value type — strong (`"abc123"`) vs weak (`W/"abc123"`).
- `ConditionalCheck` — given a request's `If-Match` / `If-None-Match` / `If-Modified-Since` / `If-Unmodified-Since` headers and the resource's current ETag + last-modified, return whether the request preconditions match.
- Helpers for serializing / parsing `Last-Modified` (uses swift-time's RFC 1123 parser).
- `ConditionalError` typed-throws enum.

Out of scope for v0.1:
- HTTP/2 push-related conditional behavior.
- Cache validation for downstream caches (different problem domain).

### Out of scope for Phase 8

- Compression encoders (v0.2 sweep) — deferred to Phase 9 as its own anchor.
- HTTP/2 / HTTP/3 protocol implementation. Out of audience.
- TLS, sockets, transport. Out of audience.
- WebSocket, SSE, server-sent events. Out of audience.
- gRPC, GraphQL, JSON-RPC. Out of audience.

## Alternatives considered

### Anchor on the crypto-adjacent wave (BLAKE3, SipHash, HKDF)

Rejected for the **fifth consecutive phase**. swift-crypto remains too entrenched. Five rejections in a row is a clear "not the right anchor for the bare-swift ecosystem at this scale" signal. Future gate retros should consider whether to formally close this wave as "indefinitely deferred" rather than re-survey it.

### Anchor on the compression-encoder v0.2 sweep

Rejected as anchor for Phase 8 because:
- 4 packages × v0.2 minor releases is anchor-shape, but the work concentrates almost entirely in swift-deflate's encoder (~1500 LOC). The other three are thin wrappers re-using the same byte-counting infrastructure.
- Mixing it with new packages (multipart / range / conditional) would dilute focus. Better as its own dedicated wave.
- **Commit to Phase 9 anchor** in this RFC's drafting: the encoder sweep is the natural follow-on to Phase 8 and unblocks `Accept-Encoding: gzip` server responses.

### Anchor on internationalization (swift-idna, swift-publicsuffix)

Rejected because:
- swift-uri's IDNA deferral is real but small. Non-ASCII hostnames are rare in server-stack code.
- swift-publicsuffix is data-heavy (~10K entries) and the use case (cookie domain validation) is niche.
- HTTP body primitives have a strictly larger audience.

### Anchor on OTLP/json variants

Rejected because:
- Audience is narrower than HTTP body primitives (only OTLP-emitting services).
- Better timing once compression encoders exist (so OTLP/json + gzip + Content-Encoding is the natural follow-on).

### Pick swift-brotli as the headline

Rejected because:
- Substantially more complex than DEFLATE (context modeling, dictionary preprocessing). Higher risk for a headline tranche.
- Better as a follow-on after compression-encoder sweep ships. May land as Phase 10+ side deliverable.

## Unresolved questions

- **Should swift-multipart support `multipart/mixed` (RFC 5322) in v0.1?** RFC 7578's form-data is structurally a subset of RFC 5322's general multipart. Resolution: v0.1 narrows to form-data semantics (the per-part `Content-Disposition: form-data; name="..."` is form-data-specific); the parser machinery generalizes if needed in v0.2.
- **Should swift-range expose a streaming slicer?** v0.1 returns `[Range<Int>]` and lets the caller slice the underlying buffer. Streaming integration with swift-bytes' BytesReader is a v0.2 nicety.
- **Does swift-conditional depend on swift-time?** Yes — `If-Modified-Since` parsing uses RFC 1123. swift-time v0.1 (Phase 5) covers this; no new commitment needed.
- **8D stretch choice.** swift-link-header is small and well-bounded; swift-brotli is substantial. Resolution: defer to execution time, lean toward swift-link-header for tractability.

## Drawbacks

- **Multipart's binary boundary detection is subtle.** Boundaries inside content are technically not allowed by RFC 7578, but real-world payloads sometimes contain them (especially when the content itself is multipart-encoded). v0.1 follows the spec strictly; consumers with non-conformant inputs see parse failures.
- **swift-conditional adds another swift-time consumer.** Already in the v0.2 Time sweep (Phase 6 closed); no new dep work.
- **No streaming API in v0.1 of any package.** Consistent with the rest of the ecosystem (msgpack/cbor/toml/json/jsonpath all single-shot in v0.1). Streaming is a v0.2-or-later commitment per package.

## Migration impact

- **No breaking changes to existing packages.** All Phase 8 work is additive new packages.
- **swift-time adoption widens.** swift-conditional becomes the 6th package depending on swift-time (after the v0.2 Time sweep added cookie/msgpack/cbor/toml/tracing-otlp/log-otlp).
- **swift-mime adoption widens.** swift-multipart depends on swift-mime to parse the boundary parameter. This makes swift-mime a load-bearing format-tier dep.

## Resolution

Accepted 2026-05-10. Phase 8 anchors on the **HTTP request/response primitives wave** with three core tranches (8A swift-multipart, 8B swift-range, 8C swift-conditional) and one stretch slot (8D — swift-link-header or swift-brotli, decided at execution time).

The compression-encoder v0.2 sweep committed in RFC-0012 is **deferred to Phase 9 as its own anchor**, formally committed here. Gate 8 retrospective will close out Phase 8 and open Phase 9 with that commitment.

Phase 8 plans may begin authoring; first package is **swift-multipart**.

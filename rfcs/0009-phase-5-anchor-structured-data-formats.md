# RFC-0009 — Phase 5 anchor: structured-data formats wave

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-09 |
| Resolution | Accepted 2026-05-09 — Phase 5 plans may begin authoring. First package: swift-msgpack. |

## Summary

Anchor Phase 5 on the **structured-data formats wave**: ship swift-msgpack (MessagePack), swift-cbor (RFC 8949), and swift-toml (TOML 1.0) as the headline packages, with swift-jsonpath (RFC 9535) or a Foundation-free date type as the optional 5D stretch. This pivots from the networking primitives tier completed in Phase 4 to the data-serialization tier — every HTTP API and config-driven service deserializes one of these formats today.

The wave reuses the Phase 2 / Phase 4 foundation (swift-bytes for binary parsers' input/output; swift-uri for URL-shaped TOML values), targets a clear adoption gap (Foundation-free MessagePack/CBOR/TOML are scarce and the existing options pull Codable or Foundation), and unblocks multiple downstream items (`Content-Type: application/msgpack` round-trip with swift-mime; OTLP/json output deferred since Phase 2; TOML-driven config in any service stack).

## Problem

[Gate 4 retrospective](../docs/gates/2026-05-09-gate-4-retrospective.md) item 5 requires "Phase 5 anchor decision recorded as an RFC" before Phase 5 plans begin. The retrospective surveyed four candidate waves (structured-data formats, time/dates foundation, crypto-adjacent, concurrency primitives); this RFC formalizes the choice.

Phases 2–4 shipped the observability tier (statsd, prometheus, hdrhistogram, ddsketch, OTLP/metrics, OTLP/traces, prometheus-metrics adapter, distributed-tracing-bridge, OTLP/logs) and the networking primitives tier (uri, mime, cookie). The format tier has five packages today (jsonpointer, dotenv, mime, cookie, uri) — none of which deserializes a *binary* payload or a *config* file. That's the next gap.

The question is: where does Phase 5's effort produce the most leverage? Structured-data formats is the answer because (a) every HTTP API and config-driven service needs at least one of MessagePack / CBOR / TOML; (b) Foundation-free options are scarce and the existing ones pull Codable; (c) the differentiation story is clearer than for crypto (vs. swift-crypto) or concurrency (vs. swift-async-algorithms).

## Proposal

### Anchor: structured-data formats wave

Phase 5 primary work targets the structured-data formats every server / API library serializes:

- **swift-msgpack** — MessagePack encoder + decoder. Binary spec (msgpack.org). Foundation-free, no Codable dep. Output type is `Bytes` for serialization; input is `Bytes`. This is the headline package — well-bounded binary spec, fits the observed-pace tranche budget.
- **swift-cbor** — Concise Binary Object Representation per RFC 8949. Sister format to MessagePack; same shape (binary, type-tagged). Foundation-free.
- **swift-toml** — TOML 1.0 parser + serializer. Text-format config files; widely used (Cargo.toml, pyproject.toml). swift-uri-aware for URL-typed values.
- **swift-jsonpath** *(stretch)* — JSONPath per RFC 9535 (Feb 2024). Pairs with Phase 1's swift-jsonpointer as the format-tier query duo. Defines a path-expression compiler + AST that consumers wire to their JSON value type; out of scope is shipping a JSON parser ourselves (that would be its own anchor).

### Tranches

| Tranche | Packages | Estimated calendar |
|---|---|---|
| 5A | swift-msgpack (MessagePack encoder + decoder) | ~1 day at observed pace; ~1–2 months at original budget |
| 5B | swift-cbor (RFC 8949 CBOR encoder + decoder) | ~1 day; ~1–2 months |
| 5C | swift-toml (TOML 1.0 parser + serializer) | ~2 days; ~2–3 months |
| 5D *(stretch)* | swift-jsonpath (RFC 9535) OR swift-time (Foundation-free date type per RFC-0010 if accepted) OR swift-yaml-lite (deliberately-narrowed YAML subset) | ~1–2 days if any |

Total Phase 5 budget: 1–2 weeks calendar at observed pace; 4–6 months at original roadmap pace.

### Why this anchor

1. **Adoption gap is clear.** MessagePack and CBOR libraries in Swift today are typically Foundation-tied or use Codable as the default serialization layer. TOML libraries exist (TOMLKit, etc.) but are Foundation-bound. Foundation-free + Bytes-in / Bytes-out + non-Codable is genuinely missing.
2. **Builds on Phase 2 + Phase 4 foundation.** swift-bytes is the natural input/output type for binary parsers — already proven across three OTLP packages. swift-varint (Phase 2) handles MessagePack's negative-int and CBOR's unsigned encoding. swift-uri (Phase 4) parses TOML's URL-shaped string values.
3. **Risk profile is known.** MessagePack and CBOR are well-bounded binary specs (~1 day each). TOML 1.0 is moderate (~2 days). Each tranche is independently scoped — no cross-package coupling like the OTLP common-types question in Phase 3. YAML deliberately deferred — a single anchor wave shouldn't carry it.
4. **Audience continuity.** Every service that consumes Phase 4's networking primitives (URLs, MIME types, cookies) eventually hits a structured-data deserialization step. The audience overlap is essentially total.
5. **Sets up downstream work.** Once MessagePack/CBOR exist: `Content-Type: application/msgpack` / `application/cbor` round-trip becomes natural with swift-mime; OTLP/json output (deferred since Phase 2) becomes feasible; TOML-driven service config (think Cargo.toml-style) becomes a one-line dep for downstream libraries.

### Tranche 5A scope sketch (subject to per-package brainstorm)

`swift-msgpack` covers the [MessagePack spec](https://github.com/msgpack/msgpack/blob/master/spec.md):

- `MsgPackValue` value type — Sendable, Equatable, Hashable. 13 cases mapping to MessagePack's type system (`nil`, `bool`, `int`, `uint`, `float32`, `float64`, `string`, `binary`, `array`, `map`, `extension`, plus the timestamp ext-type).
- Encoder: `MsgPack.encode(_ value: MsgPackValue) -> Bytes`.
- Decoder: `try MsgPack.decode(_ bytes: Bytes) throws(MsgPackError) -> MsgPackValue`.
- `MsgPackError` typed-throws enum (truncated input, invalid type byte, length-mismatch, etc.).

Out of scope for v0.1:
- Codable bridging (deliberately excluded — non-Codable is the differentiator).
- Schema validation / type coercion. The decoder produces `MsgPackValue`; mapping to user types is consumer code.
- Streaming decode of partial payloads. v0.1 takes the full Bytes payload.
- The deprecated raw-string format (pre-2013 MessagePack); decoder rejects, encoder always emits modern str format.

### Tranche 5B scope sketch

`swift-cbor` covers RFC 8949:

- `CBORValue` value type with the 8 major types (uint, nint, byte string, text string, array, map, tag, simple/float).
- Encoder + decoder mirroring swift-msgpack's shape.
- Tag handling for the canonical extension types (1: epoch-based date/time; 23: expected base64; etc.) — payload pass-through; semantic interpretation is consumer code in v0.1.
- `CBORError` typed-throws enum.

Out of scope for v0.1:
- Canonical encoding / deterministic encoding (RFC 8949 § 4.2). The encoder produces *valid* CBOR; canonical-form output is v0.2.
- CBOR Sequences (RFC 8742) — single-payload only in v0.1.
- COSE (RFC 9052) — its own anchor at minimum.

### Tranche 5C scope sketch

`swift-toml` covers TOML 1.0 (https://toml.io/en/v1.0.0):

- `TOMLValue` value type with the 6 TOML types (string, integer, float, boolean, datetime, array, table).
- Parser: `try TOML.parse(_ input: String) throws(TOMLError) -> TOMLValue`.
- Serializer: round-trip back to TOML text.
- `TOMLError` typed-throws enum.
- swift-uri integration: TOML strings that look like URLs are *not* auto-parsed in v0.1, but a `.uri()` accessor on `TOMLValue` returns `URI?` if the string parses.

Out of scope for v0.1:
- TOML 1.1 (in-progress spec).
- Datetime full RFC 3339 parsing — datetime values are preserved as raw strings in v0.1 (consistent with swift-cookie's `Expires` policy until RFC-0010 lands).
- Schema validation / decode-into-Codable.

### Out of scope for Phase 5

- A full JSON library (parser + value type). swift-jsonpointer + swift-jsonpath cover query-side; a JSON value type is its own anchor candidate (Phase 6+).
- Protobuf (full proto3 support; not just the OTLP-narrowed encoder swift-otlp-exporter ships). Its own anchor.
- HTTP/2, HTTP/3, gRPC. Out of scope as in Phase 4.
- Codable bridging across the wave. Deliberately excluded — Foundation-free + non-Codable is the differentiator.
- TLS, sockets, transport. Way out of scope.

## Alternatives considered

### Anchor on the time & dates foundation wave (swift-time, RFC 3339, RFC 1123)

Rejected as the *anchor* because:
- Single-package wave; not a wave-shape.
- Better fit as a parallel side deliverable (RFC-0010 candidate) than a phase anchor.
- Cookie v0.2 and any HTTP/2 work need it, but neither is gating Phase 5 progress.

Should be addressed in **RFC-0010** (Foundation-free date / time policy), authored in parallel with this RFC. swift-time may ship as a Phase 5 side deliverable or as the 5D stretch.

### Anchor on the crypto-adjacent wave (BLAKE3, SipHash, HKDF, X25519)

Rejected because:
- swift-crypto is too entrenched as the Apple-blessed option; differentiation requires careful positioning.
- Audience-fit is medium at best; users with crypto needs already have answers.
- Not blocking any current adopter.
- Same rejection as Phase 4 (RFC-0008). Two phases of "rejected for the same reasons" is a real signal that this isn't the next anchor.

### Anchor on concurrency primitives (AsyncSequence helpers, backpressure)

Rejected because:
- swift-async-algorithms exists with good coverage. Differentiation requires picking a specific gap (backpressure?) — too narrow for an anchor.
- Better fit as a tier 1+ side deliverable.

### Pick a single big package (full YAML, full JSON, full protobuf)

Rejected because:
- Each is a multi-week effort to do correctly. Not anchor-shape.
- YAML 1.2 in particular is multi-month for full coverage.
- Anchor waves are 3–4 packages of 1–2 days each at observed pace; single big packages slot in mid-anchor (e.g. swift-uri in Phase 4) but don't anchor on their own.

## Unresolved questions

- **Date types in TOML.** TOML's `datetime` type is core to the spec; preserving as a raw string (consistent with swift-cookie) is consistent but feels especially weak in TOML where typed datetime is part of the format identity. RFC-0010 should resolve this.
- **MessagePack timestamp extension.** Ext-type -1 carries an 8-byte or 12-byte timestamp. Decoder produces the raw bytes in v0.1; consumer parses. RFC-0010 resolution may upgrade this.
- **swift-jsonpath without a JSON value type.** JSONPath is a query language over arbitrary trees; swift-jsonpath can ship a path-expression AST + evaluator-protocol *without* a JSON parser. v0.1 ships the AST + protocol; the consumer wires evaluation to their JSON value type (or to swift-msgpack/swift-cbor). Future work: ship a `JSONValue` companion package.
- **Canonical CBOR (RFC 8949 § 4.2).** Useful for content-addressed storage and signature contexts. Deferred to v0.2 unless an early-adopter request makes it critical.

## Decision

Phase 5 anchors on the **structured-data formats wave** with three core tranches (5A swift-msgpack, 5B swift-cbor, 5C swift-toml) and one stretch slot (5D — swift-jsonpath / swift-time / swift-yaml-lite, decided at execution time based on observed pace and adopter signals). Phase 5 plans may begin authoring; first package is **swift-msgpack**.

RFC-0010 (Foundation-free date / time policy) should be authored in parallel and may inform Phase 5 5D-slot selection. RFC-0011 (GitHub Pages setup runbook) is deferred — fix-on-first-failure is operationally fine.

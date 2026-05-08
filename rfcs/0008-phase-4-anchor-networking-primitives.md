# RFC-0008 — Phase 4 anchor: networking primitives wave

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-09 |
| Resolution | Accepted 2026-05-09 — Phase 4 plans may begin authoring. First package: swift-uri. |

## Summary

Anchor Phase 4 on the **networking primitives wave**: ship swift-uri (WHATWG URL parser + serializer) as the headline package, then swift-mime (MIME parser + standard types), swift-cookie (RFC 6265 cookies), and optionally swift-http-types-extras (helpers atop Apple's swift-http-types). This pivots from the observability tier completed in Phases 2–3 to the networking tier — the next-most-requested missing piece in the swift-server world.

The wave reuses Phase 2's foundation (swift-bytes for parser inputs/outputs), targets a clear adoption gap (Foundation-free WHATWG URL has no good Swift option today), and unblocks the Phase 5 config & serialization wave (parsers benefit from URL/URI/MIME existing first).

## Problem

[Gate 3 retrospective](../docs/gates/2026-05-09-gate-3-retrospective.md) item 5 requires "Phase 4 anchor decision recorded as an RFC" before Phase 4 plans begin. The retrospective surveyed four candidate waves (networking primitives, config & serialization, crypto-adjacent, logs); this RFC formalizes the choice.

Phases 2–3 shipped the observability tier (statsd, prometheus, OTLP/metrics, OTLP/traces, prometheus-metrics adapter, distributed-tracing-bridge). The tier is complete except for the optional logs-OTLP exporter (deliberately skipped per Gate 3). Continuing to deepen observability past this point hits diminishing returns — the integration story is done.

The question is: where does Phase 4's effort produce the most leverage? Networking primitives is the answer because (a) every server library needs URL parsing, cookies, MIME types; (b) Foundation-free options are scarce; (c) the differentiation story is clearer than for crypto (vs. swift-crypto) or config/serialization (vs. Yams).

## Proposal

### Anchor: networking primitives wave

Phase 4 primary work targets the networking primitives that every HTTP server / client library needs:

- **swift-uri** — WHATWG URL parser + serializer. Foundation-free, no swift-foundation dep. Output type is `Bytes` for serialization; input is `String` or `Bytes`. This is the headline package.
- **swift-mime** — MIME type parser (`Content-Type: text/plain; charset=utf-8` etc.) + a curated catalog of standard types (`application/json`, `text/html`, `application/x-protobuf`, etc., per IANA registry).
- **swift-cookie** — RFC 6265 cookie parser + serializer (HTTP `Cookie:` and `Set-Cookie:` headers).
- **swift-http-types-extras** *(stretch)* — convenience helpers atop Apple's [`swift-http-types`](https://github.com/apple/swift-http-types) (HTTP types vended by Apple but lean — extras like header validation, common builders, content-encoding helpers).

### Tranches

| Tranche | Packages | Estimated calendar |
|---|---|---|
| 4A | swift-uri (the headline; WHATWG URL: parser + serializer + percent-encoding + IDNA-Lite) | ~1–2 days at observed pace; ~2–4 months at original budget |
| 4B | swift-mime (parser + IANA-derived catalog) | ~1 day at observed pace; ~1 month at original budget |
| 4C | swift-cookie (RFC 6265: parse + serialize, attribute parsing, expires/max-age) | ~1 day; ~1 month |
| 4D *(stretch)* | swift-http-types-extras OR (alternate) swift-jsonpath (continues format tier) OR (alternate) the deferred swift-log-otlp from Phase 3 | ~1 day if any of the three is picked |

Total Phase 4 budget: 1–2 weeks calendar at observed pace; 4–6 months at original roadmap pace.

### Why this anchor

1. **Adoption gap is clear.** WHATWG URL parsing in Swift today: Foundation's URL (legacy quirks, not WHATWG-compliant), or [`swift-url`](https://github.com/karwa/swift-url) (Apache, but quite heavy and not in the swift-server core). Foundation-free + spec-compliant + lightweight is genuinely missing.
2. **Builds on Phase 2 foundation.** swift-bytes is the right type for URL serialization output and parser input. The reader/writer cursors in `Bytes` are already idiomatic for byte-level parsers (used in swift-otlp-exporter).
3. **Risk profile is known.** Each package is independently scoped, no cross-package coupling like the OTLP common-types question in Phase 3. Tranches can ship in any order; can ship 4A alone if budget tightens.
4. **Audience continuity.** Phase 1/2/3 adopters (server-stack-adjacent users) overlap heavily with networking-primitive consumers. URL parsing in particular has been requested forever.
5. **Sets up Phase 5.** Once URL/URI/MIME exist, config/serialization parsers (TOML, YAML, MessagePack, CBOR) become more tractable — each parser can take URL-parsed configuration sources and output via Bytes.

### Tranche 4A scope sketch (subject to per-package brainstorm)

`swift-uri` covers the WHATWG URL Living Standard (https://url.spec.whatwg.org/):

- `URI` value type — Sendable, Equatable, Hashable.
- Parser: `try URI.parse(_ string: String) throws(URIError) -> URI` — implements the full WHATWG state machine.
- Serializer: `URI.serialized` and `URI.serialized(into:)` for Bytes output.
- Percent-encoding helpers (per the spec's per-component encode sets).
- IDNA-Lite for Unicode hostnames (full IDNA needs ICU; v0.1 ships ASCII-fast-path + best-effort Unicode normalization that punts to spec-compliant IDNA in v0.2).
- `URIError` typed-throws enum (parse failures: invalid scheme, invalid host, invalid port, etc.).

Out of scope for v0.1:
- Full IDNA (ICU dependency or hand-rolled mapping tables).
- Public Suffix List integration.
- Non-special URL schemes' edge cases (mailto, data, file may have minor gaps; document and fix in v0.2).
- URL pattern matching / template URLs (RFC 6570).

### Tranche 4B scope sketch

`swift-mime`:

- `MIMEType` value type with `type` (`text`), `subtype` (`plain`), `parameters` (`["charset": "utf-8"]`).
- Parser: `try MIMEType.parse(_ headerValue: String) throws(MIMEError) -> MIMEType`.
- Serializer: round-trip back to `Content-Type:` header form.
- Catalog of common types as static constants: `.applicationJSON`, `.textPlain`, `.applicationXProtobuf`, etc.
- v0.1: ~50 most common types curated from IANA. Full IANA catalog is data, not code; can be a separate package or a generated source file later.

### Tranche 4C scope sketch

`swift-cookie`:

- `Cookie` value type: name, value, path, domain, expires, max-age, secure, httpOnly, sameSite.
- Parser for `Cookie:` request header (multi-cookie) and `Set-Cookie:` response header (single cookie + attributes).
- Serializer: round-trip both header forms.
- `CookieError` typed-throws enum.
- v0.1: RFC 6265 (the practical spec). RFC 6265bis (the in-progress update) deferred.

### Out of scope for Phase 4

- Full HTTP semantics — request/response routing, body negotiation. That's swift-http-types' job (Apple's package); we provide *primitives* the routing layer composes.
- TLS, sockets, transport. Way out of scope.
- HTTP/2, HTTP/3 protocol implementation. NIO's job.
- WebSocket, SSE, server-sent events.
- gRPC, GraphQL, JSON-RPC.
- DNS resolution.

## Alternatives considered

### Anchor on the config & serialization wave (TOML, YAML, MessagePack, CBOR)

Rejected because:
- Each package is a full-spec effort (TOML 1.0 alone is weeks; YAML 1.2 is a multi-month effort to do correctly).
- Adoption-fit is "broad but each competes with established alternatives" (Yams for YAML, etc.).
- Better fit for **Phase 5** *after* networking primitives exist (parsers benefit from URL/URI for config sources).

### Anchor on the crypto-adjacent wave (BLAKE3, SipHash, HKDF, etc.)

Rejected because:
- swift-crypto is too entrenched as the Apple-blessed option; differentiation requires careful positioning.
- Audience-fit is medium at best; users with crypto needs already have answers.
- Not blocking any current adopter.

### Pick up the skipped Tranche 3C (swift-log-otlp) as the Phase 4 anchor

Rejected because:
- ~300 LOC of work; not anchor-worthy.
- Adoption-fit is medium (swift-log already has mature adapters; OTLP-log-envelope is one of several formats).
- Better as a Phase 4 side deliverable (Tranche 4D stretch) if the user wants the OTel logs signal.

### Defer Phase 4; spend the time on Phase 1/2/3 polish

Rejected because:
- Phase 1/2/3 packages are stable (CI green, test coverage comprehensive, no open bugs visible).
- The AI-assisted execution model means new-package work is currently the highest-leverage activity; polish work is endless without a forcing function.
- The Gate 3 retrospective's stop conditions did not trigger.

## Drawbacks

1. **WHATWG URL is non-trivial.** The Living Standard has hundreds of state-machine transitions, edge cases for special schemes, IDNA, percent-encoding. Even a v0.1 will be the largest single Phase 4 source file by far. Mitigation: aggressive scope-tightening for v0.1 (ASCII-fast-path IDNA; defer non-special-scheme edge cases); inline test vectors from the WHATWG test suite.
2. **IDNA without ICU is a real challenge.** Full RFC 5891 IDNA requires Unicode mapping tables (~1MB of data). v0.1 ships a punycode encoder/decoder (small, deterministic) plus ASCII-fast-path + best-effort UTS46 normalization, with a clear pathway to full IDNA in v0.2 via either an embedded mapping table or a host-system ICU shim.
3. **swift-mime catalog is data, not code.** Curating 50 types in v0.1 is fine; full IANA is ~2000 types. The package can ship as code now and grow a code-generation step later.
4. **swift-cookie has security implications.** SameSite, HttpOnly, Secure are security primitives; misimplementation has consequences. Tests must include the OWASP cookie-handling cases.
5. **swift-http-types-extras (Tranche 4D) introduces a third Apple dep.** swift-distributed-tracing-bridge already depends on swift-distributed-tracing. swift-http-types-extras would depend on swift-http-types. Two Apple deps in the ecosystem are tractable; a third sets a precedent that should probably be RFC'd first.

## Unresolved questions

These are intentionally deferred to per-package brainstorm sessions, not to the anchor RFC:

- **swift-uri input type** (`String` only? Bytes? Both?). Affects whether the parser is byte-level or unicode-level.
- **Percent-encoding API surface** (free functions, methods on URI, both?).
- **swift-mime catalog source** (hand-written constants vs. generated from IANA registry).
- **swift-cookie SameSite enum exhaustiveness** (handle Same Site as `Strict`, `Lax`, `None` enum, or as raw String?).
- **Tranche 4D selection.** swift-http-types-extras vs. swift-jsonpath vs. swift-log-otlp. Decide at start-of-4D.

## Migration impact

No migrations required. Phase 4 adds new packages; no existing v0.1+ package changes its API.

If swift-uri eventually supplants Foundation's URL in some bare-swift package (e.g., dotenv parsing of `URL=...` lines, or some not-yet-built networking-touching package), that package would minor-bump (pre-1.0 convention). Not in scope for this anchor.

## Resolution

- **Accepted**
- Date: 2026-05-09
- Notes: Closes Gate 3 item 5. Phase 4 plans begin with swift-uri (Tranche 4A). IDNA scope decision (ASCII-fast-path + best-effort UTS46 in v0.1, full IDNA in v0.2) is the key open scoping question for swift-uri's per-package brainstorm. Tranche 4D selection deferred to start-of-4D per RFC.

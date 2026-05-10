# RFC-0011 — Phase 6 anchor: JSON tier + downstream v0.2 sweep

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-10 |
| Resolution | Accepted 2026-05-10 — Phase 6 plans may begin authoring. First package: swift-json. |

## Summary

Anchor Phase 6 on the **JSON tier**: ship swift-json (RFC 8259 value type + parser + serializer) as the headline package, then swift-jsonpath (RFC 9535) and swift-jsonpatch (RFC 6902); swift-jsonschema-lite as the optional 6D stretch. Run the **downstream v0.2 adoption sweep** committed in RFC-0010 as a parallel work-stream during Phase 6 execution: cookie / tracing-otlp / log-otlp / msgpack / cbor / toml each get a minor bump that wires in `Time` types additively.

The wave reuses the Phase 2 + Phase 5 foundation (swift-bytes for parser input/output; swift-time for date-bearing JSON values when the user opts in), targets the largest-audience format gap left in the ecosystem (every HTTP API serializes JSON), and frees swift-jsonpointer (Phase 1) from its current "BYO JSON value type" limitation.

## Problem

[Gate 5 retrospective](../docs/gates/2026-05-10-gate-5-retrospective.md) item 5 requires "Phase 6 anchor decision recorded as an RFC" before Phase 6 plans begin. The retrospective surveyed five candidate waves (JSON tier, crypto-adjacent, concurrency, downstream v0.2 sweep alone, internationalization); this RFC formalizes the choice.

Phases 2–5 shipped the observability tier (9 packages), the networking primitives tier (3 packages), the structured-data formats tier (4 packages, both binary and text), and reactivated the foundation tier with swift-time. Every Phase 1–5 audience consumer eventually hits a JSON serialization step. Foundation's `JSONSerialization` is Codable-backed and Foundation-bound; swift-foundation has the same. A clean `JSONValue` enum + Bytes-aware parser is the largest unfilled niche.

Concurrently, RFC-0010's migration plan committed five Phase 4–5 packages to v0.2 minor releases that wire in `Time` types. Carrying that debt without execution leaves the ecosystem in an unfinished state. Phase 6 picks it up as a parallel work-stream — not as a tranche, because v0.2 bumps don't fit the "new packages" anchor shape, but as a documented commitment alongside the JSON wave.

## Proposal

### Anchor: JSON tier

Phase 6 primary work targets the JSON-format triangle that every HTTP-adjacent service needs:

- **swift-json** — RFC 8259 value type (`JSONValue` enum: `null`, `bool`, `integer`, `double`, `string`, `array`, `object`) + parser + serializer. Foundation-free, non-Codable, Bytes-aware. Splits integers and doubles to preserve `Int64` precision (consistent with swift-msgpack / swift-cbor's split). This is the headline package.
- **swift-jsonpath** — RFC 9535 (Feb 2024). Path-expression compiler + AST + evaluator over `JSONValue`. Pairs with Phase 1's swift-jsonpointer as the format-tier query duo (jsonpointer addresses single nodes; jsonpath returns sets).
- **swift-jsonpatch** — RFC 6902 (`add` / `remove` / `replace` / `move` / `copy` / `test` operations) applied to `JSONValue`. Closes the JSON-mutation triangle.
- **swift-jsonschema-lite** *(stretch)* — a deliberately-narrowed subset of JSON Schema 2020-12 (type / required / properties / items / enum / const). Out of scope: full validator, $ref resolution, format keyword, custom keyword extensions. Exists to give the ecosystem a "validate this JSON against this shape" primitive without committing to the full spec.

### Tranches

| Tranche | Packages | Estimated calendar |
|---|---|---|
| 6A | swift-json (JSONValue + RFC 8259 parser + serializer) — the headline | ~1–2 days at observed pace |
| 6B | swift-jsonpath (RFC 9535 path queries over JSONValue) | ~1 day |
| 6C | swift-jsonpatch (RFC 6902 patch operations on JSONValue) | ~1 day |
| 6D *(stretch)* | swift-jsonschema-lite (deliberately-narrowed JSON Schema 2020-12 subset) OR continue the v0.2 sweep | ~1–2 days if any |

### Parallel work-stream: downstream v0.2 adoption sweep

Per RFC-0010's migration plan, the following minor releases ship during Phase 6 execution (interleaved with the tranches, not blocking them):

| Package | v0.2 addition |
|---|---|
| swift-cookie | `Cookie.expiresAt: Time.Calendar?` (parsed); `expires: String?` retained for raw-text round-trip. |
| swift-tracing-otlp | Convenience initializers on `OTLP.Span` accepting `Time.Instant` for time fields. |
| swift-log-otlp | Same — `OTLP.LogRecord` initializers accepting `Time.Instant`. |
| swift-msgpack | `MsgPackValue.timestamp(Time.Instant)` case for ext-type −1; raw `.ext(-1, Bytes)` round-trips for compatibility. |
| swift-cbor | `CBOR.encodeDate(_:) -> Bytes` / `CBOR.decodeDate(_ tagged: CBORValue) -> Time.Instant?` helpers wrapping tags 0 (RFC 3339 string) and 1 (epoch number). |
| swift-toml | `TOMLValue.datetime(Time.Calendar)` typed accessor alongside the raw-string case (the `datetime(String)` case is renamed to `rawDatetime(String)` if and only if a clean migration path exists; otherwise both forms coexist). |

Each migration is **additive** — new cases / new methods alongside the old. No breaking changes; consumers opt in. Estimated effort: ~30 minutes per package, ~3 hours total.

The sweep is committed in this RFC. Execution order is flexible (interleave with 6A/6B/6C as cognitively convenient).

Total Phase 6 budget: 1–2 weeks calendar at observed pace; 4–6 months at original roadmap pace.

### Why this anchor

1. **Adoption gap is largest of all surveyed candidates.** Every HTTP API and every observability consumer of the OTLP packages eventually writes JSON. Foundation's `JSONSerialization` is Codable-bound and Foundation-bound. swift-foundation matches it. Foundation-free + non-Codable + `Bytes`-aware is genuinely missing.
2. **Pairs with existing tier.** swift-jsonpointer (Phase 1) has waited four phases for a `JSONValue` type to operate on. swift-jsonpath (RFC 9535, Feb 2024) is the natural sibling. swift-jsonpatch (RFC 6902) closes the mutation triangle.
3. **Builds on Phase 2 + Phase 5 foundations.** swift-bytes is the natural input/output type. swift-time provides the optional date type for tag-0 / RFC-3339 helpers.
4. **Differentiation is real.** Foundation's API returns `Any`; consumers cast at runtime. swift-json's `JSONValue` enum is exhaustively-checkable at compile time (Swift's switch-coverage). Same advantage swift-msgpack and swift-cbor have over `JSONSerialization`-style APIs.
5. **Risk profile is known.** swift-json is moderate (RFC 8259 grammar; smaller than TOML 1.0). swift-jsonpath is a path-expression AST + evaluator (~1 day). swift-jsonpatch is straightforward (six op types; trivial dispatch). swift-jsonschema-lite is the wildcard; defer or scope narrowly per execution-time judgment.
6. **Sets up Phase 7+.** Once swift-json exists: swift-otlp-exporter / swift-tracing-otlp / swift-log-otlp can ship JSON-mode encoders (long-deferred). swift-msgpack / swift-cbor ↔ JSON helpers become trivial. JSON-Schema-based config loading becomes feasible.

### Tranche 6A scope sketch

`swift-json` covers RFC 8259:

- `JSONValue` value type — Sendable, Equatable. 7 cases:
  - `null`
  - `bool(Bool)`
  - `integer(Int64)` — whole numbers without fractional or exponent parts
  - `double(Double)` — numbers with fractional / exponent parts
  - `string(String)`
  - `array([JSONValue])`
  - `object([Member])` — ordered insertion-preserving (matches swift-msgpack/cbor/toml pattern)
- `Member` struct: `key: String`, `value: JSONValue`.
- `JSON.parse(_ String) throws(JSONError) -> JSONValue` — strict UTF-8 input; standard JSON whitespace tolerance; full `\uXXXX` and surrogate-pair handling.
- `JSON.serialize(_ value: JSONValue) -> String` — round-trip emitter; `\u00XX` for control chars; deliberately *non-canonical* (insertion order preserved over lexicographic order, since `JSONValue` carries order intentionally).
- `JSONError` typed-throws enum with line/column metadata where available.

Out of scope for v0.1:
- `Codable` bridging — deliberately excluded (same differentiator as swift-msgpack/cbor/toml).
- Streaming partial decode. v0.1 takes a single full input.
- JSON5 / relaxed JSON / comments. Strict RFC 8259 only.
- Pretty-printing / configurable indent. Round-trip is compact form.
- Number-precision preservation beyond Int64 / Double. Bignum / decimal is v0.2 if requested.

### Tranche 6B scope sketch

`swift-jsonpath` covers RFC 9535:

- `JSONPathExpression` — compiled path AST.
- `JSONPath.parse(_:)` — compile a path expression like `$.store.book[?(@.price < 10)].title`.
- `JSONPath.query(_ expression: JSONPathExpression, on root: JSONValue) -> [JSONValue]` — returns the matching nodeset.
- Filter expressions (`?(...)`), array slicing (`[:5]`), wildcards (`*`), descendant operator (`..`), bracket and dot notation.

Out of scope for v0.1:
- Function extensions beyond the four mandatory ones (`length()`, `count()`, `match()`, `search()`).
- Custom function registration.
- Performance optimization — clean correctness first.

### Tranche 6C scope sketch

`swift-jsonpatch` covers RFC 6902:

- `JSONPatch` value type — array of operations.
- `JSONPatchOperation` enum — `add`, `remove`, `replace`, `move`, `copy`, `test` with payloads.
- `JSONPatch.apply(to: inout JSONValue) throws(JSONPatchError)` — atomic application (all-or-nothing rollback if any operation fails).
- Round-trip parser/serializer for the patch document itself (a JSON array per the RFC).

Out of scope for v0.1:
- JSON Merge Patch (RFC 7396). Different format; could ship as separate package if asked.

### Out of scope for Phase 6

- Full JSON Schema 2020-12 (the stretch package is a deliberately-narrowed subset).
- JSON-RPC. Different problem; out of audience for this phase.
- Full streaming JSON parser. Single-payload only.
- BSON / Smile / UBJSON. Other binary formats; Phase 7+ if requested.
- Codable bridging across the wave. Same exclusion as Phase 5.

## Alternatives considered

### Anchor on the crypto-adjacent wave (BLAKE3, SipHash, HKDF)

Rejected for the third consecutive phase. swift-crypto is too entrenched; the differentiation argument is harder to make than for any other surveyed wave. Three rejections in a row is a strong signal this is not the right anchor for the bare-swift ecosystem at this scale. May reconsider if a specific consumer with a hard "no swift-crypto" constraint surfaces.

### Anchor on the downstream v0.2 sweep alone

Rejected because:
- Not anchor-shape. Anchor waves ship new packages; v0.2 bumps are migration debt.
- Six bumps × 30 min ≈ 3 hours total. Too small to fill a phase.
- Better as a parallel work-stream within a larger anchor (which is what this RFC commits).

### Anchor on internationalization (swift-idna, swift-publicsuffix)

Rejected because:
- Niche audience. swift-uri's IDNA deferral is a real gap but a small one — non-ASCII hostnames are rare in server-stack code.
- Better fit as a Phase 7+ side deliverable when a specific consumer asks.

### Pick a single big package (swift-yaml-full, full JSON Schema)

Rejected because:
- Each is multi-week. Anchor waves are 3–4 packages of 1–2 days each.
- swift-yaml-lite was already considered as RFC-0009's 5D-stretch and skipped; the size argument hasn't changed.
- swift-jsonschema-lite (the 6D stretch) is the right scope: a deliberately-narrowed subset, not the full spec.

## Unresolved questions

- **Number precision in `JSONValue`.** Splitting `integer(Int64)` and `double(Double)` covers 99% of cases but loses precision for big-integer JSON (`2^63`+ unsigned). Resolution: v0.1 truncates / loses precision for values outside `Int64`'s range and falls back to `double`. v0.2 may add a `bignum(String)` case if a real consumer needs it.
- **JSONPath function extensions beyond the four mandatory ones.** RFC 9535 allows custom function registration. v0.1 ships only the four mandatory; v0.2 may add a registration API if asked.
- **JSON Merge Patch (RFC 7396).** Skip in v0.1 of swift-jsonpatch; ship as a separate package if requested.
- **`object([Member])` vs `object([String: JSONValue])`.** Insertion-order preservation is consistent with swift-msgpack / swift-cbor / swift-toml. Costs `O(n)` lookup. Most JSON consumers iterate; the rare lookup-heavy consumer can build their own dictionary view.
- **swift-time integration in v0.1.** swift-json v0.1 does NOT take a Time dep; date semantics belong on the JSON Schema layer (or in CBOR's encodeDate helpers) rather than `JSONValue` itself. v0.2 may add date-aware helpers if asked.

## Drawbacks

- **JSON is the most-implemented format on Earth.** Differentiation requires consistent positioning ("Foundation-free, non-Codable, value-type-first") and execution discipline. Easy to over-scope.
- **swift-jsonpath's RFC 9535 is recent (Feb 2024) and complex.** Filter-expression grammar has subtle edge cases. v0.1 must scope conservatively.
- **swift-jsonschema-lite's "lite" boundary is judgment-dependent.** Different consumers want different subsets. The 6D-stretch nature lets us defer if scoping isn't clean.

## Migration impact

- **Each Phase 4–5 package's v0.2** adopts swift-time additively (no breaking changes). Migration plan from RFC-0010 is unchanged.
- **swift-jsonpointer (Phase 1) is unchanged in v0.1**, but its v0.2 may adopt a thin wrapper over `JSONValue` to make the common case (parse JSON, evaluate pointer) a one-liner. Defer to v0.2 brainstorm.
- **No breaking changes anywhere.** All Phase 6 work is additive across the ecosystem.

## Resolution

Accepted 2026-05-10. Phase 6 anchors on the **JSON tier** with three core tranches (6A swift-json, 6B swift-jsonpath, 6C swift-jsonpatch) and one optional stretch slot (6D — swift-jsonschema-lite or continued v0.2 sweep, decided at execution time). The **downstream v0.2 adoption sweep** committed in RFC-0010 runs as a parallel work-stream; six minor releases ship interleaved with the tranches.

Phase 6 plans may begin authoring; first package is **swift-json**.

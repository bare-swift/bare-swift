# RFC-0010 — Foundation-free date / time policy

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-10 |
| Resolution | Accepted 2026-05-10 — `swift-time` will ship as a Phase 5 5D-stretch or Phase 6 foundation-tier package; downstream packages keep raw-string / `UInt64`-nanos representations until then. |

## Summary

Ship a Foundation-free **`swift-time`** foundation-tier package with three public types — ``Instant`` (a wall-clock instant in UTC), ``Duration`` (signed time span), and ``Calendar`` (a minimal year/month/day breakdown for date arithmetic) — plus parsers and serializers for **RFC 3339** (the format CBOR tag 0, TOML, and modern HTTP use) and **RFC 1123** (the legacy HTTP date format used by `Cookie: Expires` and many RFC 7231-era headers). Defer leap-second handling, full timezone-database support, and locale-aware formatting to v0.2+.

In the meantime, downstream packages that already need date semantics (swift-cookie's `Expires`, swift-distributed-tracing-bridge's `TracerInstant`, swift-msgpack's timestamp ext-type, swift-cbor's tags 0 and 1, swift-toml's `datetime`) keep their current Foundation-free shape: raw `String` for text-format dates, raw `UInt64` Unix nanoseconds for binary-format instants. v0.2 of each will adopt `swift-time` types in an additive, source-stable way.

## Problem

The Foundation-free date gap is a confirmed two-incident pattern documented in the [Gate 4 retrospective](../docs/gates/2026-05-09-gate-4-retrospective.md) and surfaced again in Phase 5:

| Incident | Phase | Symptom |
|---|---|---|
| `swift-distributed-tracing-bridge` | 3B | Custom `TracerInstant` defined per-package because no shared instant type existed. |
| `swift-cookie` | 4C | `Expires` attribute preserved as raw `String`; date parsing deferred to v0.2 with no concrete plan. |
| `swift-msgpack` | 5A | Ext-type −1 (timestamp) preserved as raw `Bytes`; semantic interpretation deferred. |
| `swift-cbor` | 5B | Tag 0 (RFC 3339 datetime) and tag 1 (epoch number) pass through with raw inner value. |
| `swift-toml` | 5C *(upcoming)* | TOML's `datetime` type is core to the spec; a raw-string fallback is especially weak in TOML where typed datetime is part of the format identity. |

Every networking / serialization package in the ecosystem hits this. RFC-0001 (no Foundation) holds, but the cost is real: each package carries its own ad-hoc representation, and downstream consumers can't compare instants across packages without re-implementing parsers.

This RFC commits the policy. Either ship a foundation-tier date type so future packages stop re-inventing, or commit to the per-package status quo permanently. The former is the right choice because (a) the gap is real and recurring, (b) Apple's `swift-foundation` doesn't satisfy RFC-0001, (c) every other Phase 1 foundation tier (bytes, varint, hex, base64) followed the same "ship the missing primitive" pattern with success.

## Proposal

### Anchor: ship `swift-time` as a foundation-tier package

```swift
import Time

// Instants are wall-clock UTC, nanosecond resolution, signed (Int64) so
// pre-Unix-epoch values are representable. Range: roughly ±292 years
// either side of 1970.
public struct Instant: Sendable, Equatable, Hashable, Comparable {
    /// Nanoseconds since the Unix epoch (1970-01-01T00:00:00Z), Int64-bound.
    public var nanosecondsSinceEpoch: Int64

    public init(nanosecondsSinceEpoch: Int64)
    public static let unixEpoch: Instant
}

public struct Duration: Sendable, Equatable, Hashable, Comparable {
    /// Nanoseconds; signed to allow negative spans.
    public var nanoseconds: Int64

    public init(nanoseconds: Int64)
    public static let zero: Duration

    // Convenience constructors
    public static func seconds(_ s: Int64) -> Duration
    public static func milliseconds(_ ms: Int64) -> Duration
    public static func microseconds(_ us: Int64) -> Duration
}

// Arithmetic
public func + (lhs: Instant, rhs: Duration) -> Instant
public func - (lhs: Instant, rhs: Instant) -> Duration

/// A *broken-down* civil date in UTC. Used for text-format
/// serialization (RFC 3339, RFC 1123) and basic calendar-arithmetic
/// helpers. NOT a timezone-aware calendar — that's deferred to v0.2.
public struct Calendar: Sendable, Equatable, Hashable {
    public var year: Int
    public var month: Int      // 1..12
    public var day: Int        // 1..31
    public var hour: Int       // 0..23
    public var minute: Int     // 0..59
    public var second: Int     // 0..59 (no leap-second support in v0.1)
    public var nanosecond: Int // 0..999_999_999
    public var offsetSeconds: Int  // signed; 0 for UTC

    public func toInstant() -> Instant
    public static func from(_ instant: Instant, offsetSeconds: Int = 0) -> Calendar
}

// Parsers and serializers
public enum RFC3339 {
    public static func parse(_ s: String) throws(TimeError) -> Calendar
    public static func serialize(_ c: Calendar) -> String
}

public enum RFC1123 {
    public static func parse(_ s: String) throws(TimeError) -> Calendar
    public static func serialize(_ c: Calendar) -> String
}

public enum TimeError: Error, Equatable, Sendable {
    case invalidFormat(String)
    case outOfRange(field: String)
    case invalidOffset(String)
}
```

### Key design choices

1. **`Int64` nanoseconds, not `UInt64`.** `swift-distributed-tracing-bridge`'s `TracerInstant` is `UInt64` because span events post-date the Unix epoch by definition. `Cookie: Expires` and TOML datetimes can predate it. Signed gives ±292 years either side of 1970, plenty for HTTP / config use cases.
2. **No timezone database in v0.1.** `Calendar` carries an `offsetSeconds` field (e.g. `-18000` for `-05:00`); applying timezone *names* (`America/New_York`) requires the IANA zone database which would balloon the package. v0.1 lets parsers preserve the offset literally; arithmetic stays in UTC.
3. **No leap seconds in v0.1.** UTC arithmetic ignores them. Real-world cookie / OTLP / HTTP date use cases are not leap-second-sensitive.
4. **No `Codable` conformance.** Same differentiator as the rest of the bare-swift ecosystem.
5. **`Duration` is bare-swift's, not stdlib's.** Swift's stdlib has `Swift.Duration` (5.7+), but it's tied to picosecond precision and `ContinuousClock`/`SuspendingClock` interop that doesn't match wall-clock UTC use cases. Using a custom `Time.Duration` keeps the API focused. Conversions to/from stdlib `Swift.Duration` are a v0.2-stretch nicety.
6. **`Calendar` is minimal.** Year/month/day breakdown is enough to serialize RFC 3339 / RFC 1123. Day-of-week, week-number, locale-aware arithmetic, weekday names, etc. are out of scope.

### Migration plan

Once `swift-time` ships, existing packages adopt it additively in their next minor release:

| Package | Today | After `swift-time` |
|---|---|---|
| swift-cookie | `expires: String?` | Add `expiresAt: Time.Calendar?` (parsed) alongside; `expires` stays for raw-text round-trip. |
| swift-tracing-otlp | `startTimeUnixNano: UInt64` | No change to wire fields; add convenience initializer accepting `Time.Instant`. |
| swift-msgpack | timestamp ext as raw `Bytes` | Add `MsgPackValue.timestamp(Time.Instant)` case; raw `.ext(-1, Bytes)` round-trips for compatibility. |
| swift-cbor | tag 0/1 raw inner value | Add convenience `CBOR.encodeDate(_:)` / `CBOR.decodeDate(...)` helpers using `Time.Calendar`. |
| swift-toml | `datetime` as raw `String` | Add `TOMLValue.datetime(Time.Calendar)` case; raw-string accessor preserved for fall-through. |

All migrations are additive (new cases / methods alongside the old) — no breaking changes; consumers opt in.

### Tranche placement

Two viable slots:

- **5D stretch slot.** RFC-0009 lists `swift-time` as one of the three 5D-stretch options. Picking it ships the foundation in Phase 5, immediately unblocking swift-toml v0.1's datetime handling.
- **Phase 6 foundation tier.** If 5D goes to swift-jsonpath or swift-yaml-lite, swift-time slips to Phase 6. Phase-5 packages stay at raw-representation v0.1; v0.2 adoption happens in Phase 6+.

The RFC does **not** mandate the slot — that's an execution-time decision per RFC-0009. The RFC commits *what* swift-time looks like and *when* downstream packages adopt it. The *when* is "next minor release after swift-time ships", regardless of which phase that lands in.

## Alternatives considered

### Per-package raw representations forever

Rejected because:
- The two-incident pattern is now five-incident with TOML on the horizon. Refusing a shared type guarantees re-invention in every future format package.
- Cross-package date comparison is impossible without a shared type. Want to filter trace spans by cookie expiry? Re-implement RFC 1123 → RFC 3339 conversion yourself.
- Foundation's `Date` exists for exactly this reason. The only reason not to use it is the RFC-0001 ban. A bare-swift equivalent costs ~1 day to ship and amortizes across the entire ecosystem.

### Adopt Apple's `swift-foundation` as a transitive dep

Rejected because:
- Violates RFC-0001's "no Foundation in any form, transitively" guarantee. swift-foundation is Foundation under a different name.
- The Phase 4 audience (Foundation-free server libraries) chose bare-swift specifically to avoid this dep. Pulling it in would be a regression.

### Use stdlib `Swift.Duration` and skip `Time.Duration`

Rejected because:
- Stdlib `Duration` is `(Int64, Int64)` picosecond resolution — overkill for ms/μs/ns wall-clock work, and the picoseconds field is dead weight in our serialization paths.
- Stdlib `Duration` has no associated `Instant` type that's wall-clock UTC; the `Clock` protocol's `Instant` is monotonic-clock-bound (`ContinuousClock.Instant` is uptime-relative).
- Reinventing here is genuinely cheap (a wrapper around `Int64`), and the API surface stays focused.

A v0.2 nicety: ship interop functions converting `Time.Duration` ↔ `Swift.Duration` when the consumer wants stdlib-clock work.

### Ship the parsers without the value types (parser-only package)

Rejected because:
- A parser without a value type returns either a tuple `(year, month, day, hour, min, sec, offset)` or a `String`-keyed dictionary. Consumers reinvent the value type anyway.
- The two-incident gap is specifically a value-type gap; parsing-only doesn't close it.

### Include full IANA timezone database

Rejected because:
- Tens of MB of compiled tables. Not foundation-tier scope.
- The v0.1 use cases (cookie expiry, OTLP timestamps, TOML config datetimes) all carry their offset on the wire; no zone-name resolution needed.
- swift-time can grow to add an IANA-DB companion package later without breaking v0.1.

## Drawbacks

- **One more package the ecosystem must maintain.** Adds to surface area but addresses a real recurring gap.
- **Adoption cost on existing packages.** swift-cookie / swift-tracing-otlp / etc. each take a minor-version bump to wire in. All migrations are additive, but they're work.
- **Tying ourselves to a custom `Duration` type means stdlib interop is one conversion away.** v0.2 fixes this; v0.1 callers using stdlib clocks bridge manually.

## Unresolved questions

- **Should `Calendar` distinguish "local datetime" (no offset known) from "datetime with offset = 0"?** TOML's grammar has separate types for "local-datetime" and "offset-datetime". Resolution: add an optional `offsetSeconds: Int?` (nil = local, value = offset-aware). Defer the final type shape to swift-time v0.1's brainstorm.
- **What happens when an RFC 3339 string has fractional seconds beyond nanosecond precision?** Resolution: truncate (don't error). Document it; revisit if a real consumer hits the edge.
- **Should `Time.Instant` and stdlib `ContinuousClock.Instant` be conceptually distinct or aliased?** Resolution: distinct. `Time.Instant` is wall-clock UTC; `ContinuousClock.Instant` is monotonic uptime. Different semantics, different types. v0.2 may add a helper `Time.Clock.now()` returning `Time.Instant`.

## Migration impact

- **No breaking changes to existing packages.** Adoption is additive; raw-string / `UInt64`-nanos representations remain in place and round-trip alongside the new typed APIs.
- **swift-time depends only on stdlib.** No dep cycle with swift-bytes or others. Foundation-tier placement.
- **Downstream packages should stage adoption** at their next minor release, not their next major. v0.1 → v0.2 of each is the natural slot.

## Resolution

Accepted 2026-05-10. swift-time will ship as either Phase 5 5D-stretch or Phase 6 foundation-tier; the slot decision happens at execution time per RFC-0009. Until swift-time ships, downstream packages keep their current raw representations — no scrambling to retrofit. After swift-time ships, each downstream package adopts in its next minor release additively.

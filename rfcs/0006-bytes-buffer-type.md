# RFC-0006 — Bytes / span-like buffer type

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-07 |
| Resolution | Accepted 2026-05-07 — unblocks Tranche 2A; swift-bytes is the first Phase 2 package to author. |

## Summary

bare-swift ships its own `Bytes` value type as the canonical byte-buffer used by binary-format packages, paired with `BytesReader` / `BytesWriter` cursor types. Foundation-free, owned storage, no dependency on swift-nio, no wait on stdlib `RawSpan` stabilization. Conversion bridges to `[UInt8]` and (eventually) `Span<UInt8>` are first-class. swift-otlp-exporter (Tranche 2B) and any future serialization-wave package use it as their byte-IO seam.

## Problem

RFC-0005 pinned the Phase 2 anchor on the observability wave; swift-otlp-exporter (Tranche 2B) is the most ambitious item in that wave. OTLP is a protobuf wire format, which means the package's hot path is byte-by-byte serialization: write a varint, write a length-prefixed string, write a fixed-width float64, read a tag, etc.

Phase 1 packages used `[UInt8]` as their byte type. That worked for hashes, encoders, and small format converters, but it has friction at scale:

- **No clean cursor model.** Every Phase 1 decoder threaded `offset: Int` through call after call. For a complex format (protobuf, CBOR, MessagePack), this gets ugly fast.
- **No standard primitives.** Reading a big-endian UInt32 from `[UInt8]` is a five-line snippet that gets re-implemented in every package. We have it inlined in swift-xxhash, swift-uuid, swift-crc, and swift-jsonpointer; same logic, slightly different shape, four places to keep correct.
- **Conversion costs.** Every package that reads `Sequence<UInt8>` has fast/slow path machinery (`withContiguousStorageIfAvailable` then `Array` fallback). That logic should live in one place.
- **No buffer reuse story.** A protobuf serializer building up a payload byte-by-byte ideally pre-sizes an output buffer and writes into it. `[UInt8]` does this fine but exposes the underlying allocation choices to the caller.

The Swift ecosystem also has multiple existing answers — `swift-nio.ByteBuffer`, `Foundation.Data`, the evolving stdlib `Span<UInt8>` / `RawSpan` — none of which match the bare-swift posture (Foundation-free, dependency-light, simple).

## Proposal

### `Bytes` — owned, mutable, value-typed

```swift
public struct Bytes: Sendable, Hashable, ExpressibleByArrayLiteral, RandomAccessCollection {
    /// Internal contiguous storage. Public for inspection only;
    /// mutate via the documented operations.
    public internal(set) var storage: ContiguousArray<UInt8>

    public init()
    public init(_ array: [UInt8])
    public init(_ array: ContiguousArray<UInt8>)
    public init(_ sequence: some Sequence<UInt8>)
    public init(repeating: UInt8, count: Int)
    public init(reservingCapacity: Int)
    public init(arrayLiteral: UInt8...)

    // MARK: - RandomAccessCollection
    public var count: Int { get }
    public var isEmpty: Bool { get }
    public var startIndex: Int { get }
    public var endIndex: Int { get }
    public subscript(index: Int) -> UInt8 { get set }
    public subscript(range: Range<Int>) -> Bytes { get }   // copy-out

    // MARK: - Mutation
    public mutating func append(_ byte: UInt8)
    public mutating func append(contentsOf: some Sequence<UInt8>)
    public mutating func append(_ other: Bytes)
    public mutating func reserveCapacity(_ n: Int)
    public mutating func removeAll(keepingCapacity: Bool = false)

    // MARK: - Borrowing
    public func withUnsafeBufferPointer<R>(_ body: (UnsafeBufferPointer<UInt8>) throws -> R) rethrows -> R
    public mutating func withUnsafeMutableBufferPointer<R>(_ body: (inout UnsafeMutableBufferPointer<UInt8>) throws -> R) rethrows -> R

    // MARK: - Convert
    public var array: [UInt8] { get }
    public func toArray() -> [UInt8]
}
```

Storage is `ContiguousArray<UInt8>` — one indirection less than `[UInt8]` (no bridging to NSArray on Apple platforms), strictly equivalent in API. Internal representation is allowed to evolve; the storage property is exposed for consumers that need it (e.g., bridging to swift-nio downstream packages).

### `BytesReader` — sequential decode cursor

```swift
public struct BytesReader: ~Copyable {
    private let bytes: Bytes
    public private(set) var offset: Int

    public init(_ bytes: Bytes)

    // MARK: - Single-byte
    public mutating func readByte() throws(BytesError) -> UInt8
    public mutating func peekByte() throws(BytesError) -> UInt8

    // MARK: - Fixed-width integers
    public mutating func readUInt16LE() throws(BytesError) -> UInt16
    public mutating func readUInt32LE() throws(BytesError) -> UInt32
    public mutating func readUInt64LE() throws(BytesError) -> UInt64
    public mutating func readUInt16BE() throws(BytesError) -> UInt16
    public mutating func readUInt32BE() throws(BytesError) -> UInt32
    public mutating func readUInt64BE() throws(BytesError) -> UInt64

    // MARK: - Bulk
    public mutating func readBytes(_ n: Int) throws(BytesError) -> Bytes
    public mutating func skip(_ n: Int) throws(BytesError)

    public var remaining: Int { get }
    public var isAtEnd: Bool { get }
}
```

`~Copyable` because the reader's `offset` is positional state; copying it would create two cursors that silently diverge. The non-copyable choice forces explicit reset semantics if the user actually wants two cursors.

### `BytesWriter` — sequential encode cursor

```swift
public struct BytesWriter {
    public private(set) var bytes: Bytes
    public init(reservingCapacity: Int = 0)

    public mutating func writeByte(_ byte: UInt8)
    public mutating func writeUInt16LE(_ value: UInt16)
    public mutating func writeUInt32LE(_ value: UInt32)
    public mutating func writeUInt64LE(_ value: UInt64)
    public mutating func writeUInt16BE(_ value: UInt16)
    public mutating func writeUInt32BE(_ value: UInt32)
    public mutating func writeUInt64BE(_ value: UInt64)
    public mutating func writeBytes(_ bytes: Bytes)
    public mutating func writeBytes(_ array: [UInt8])
    public mutating func writeBytes(_ array: some Sequence<UInt8>)

    public consuming func finish() -> Bytes
}
```

`finish()` is `consuming` so the writer can't be reused after extracting the buffer; this prevents accidental aliasing between a "finished" `Bytes` and further writer mutations.

### `BytesError` — typed throws

```swift
public enum BytesError: Error, Equatable, Sendable {
    /// Read advanced past the end of the buffer.
    case truncated
}
```

Exactly one case in v0.1. Reader errors are uniformly "out of bytes"; if a subsequent package (e.g., swift-cbor) needs richer reader errors, it wraps in its own error type rather than expanding `BytesError`.

### Bridging

- **From `[UInt8]`:** `Bytes(array)`. Allocates one contiguous storage buffer.
- **To `[UInt8]`:** `bytes.array` (computed; copies into a fresh `[UInt8]` storage, since `[UInt8]` and `ContiguousArray<UInt8>` have separate storage representations).
- **To `Span<UInt8>` / `RawSpan`:** Once stdlib stabilizes those types, add `var span: Span<UInt8>` / `var rawSpan: RawSpan { get }` accessors. Out of v0.1 scope.
- **To `swift-nio.ByteBuffer`:** Out of scope. Adopters write their own one-line bridge if needed (`ByteBuffer(bytes: bytes.storage)`); we don't take a swift-nio dependency to ship that bridge in-package.

### Out of scope for v0.1

- Mutable slicing into a sub-region of an existing `Bytes` without copying (i.e., a `MutableBytesView`). Stdlib's `MutableSpan` is the right home for this; we'll add bridges when it stabilizes.
- Variable-length integer encoding/decoding (varint / LEB128). Composes with swift-varint; not part of `Bytes` itself.
- UTF-8 view over `Bytes`. Composes with `String(decoding: bytes.storage, as: UTF8.self)`; not built-in.
- Arena / pooled allocator. Future optimization.
- Big-endian / little-endian generic abstraction. Explicit per-method (`readUInt32LE` / `readUInt32BE`) beats a generic `Endianness` parameter for clarity.

## Alternatives considered

**Adopt `swift-nio.ByteBuffer` directly.** Rejected. Pulling in swift-nio as a dependency contradicts the "no swift-metrics dependency" posture we took for swift-prometheus and similarly contradicts the bare-swift thesis. swift-nio is also actively evolving its own ByteBuffer story; coupling our entire byte-IO seam to its release cadence and design choices is more risk than benefit. Adopters who already use swift-nio can write a one-line bridge without our help.

**Wait for stdlib `Span<UInt8>` / `RawSpan`.** Rejected. `Span` is non-owning by design — it's a borrowed view, not a storage type. Even when stable, we'd still need `[UInt8]` (or equivalent) for storage and `Span<UInt8>` for transit. That's the same two-type situation as our `Bytes` + `BytesReader`/`BytesWriter` shape, except we'd be deferring the design indefinitely. Better to ship a shape we control now and adopt `Span` bridges when they land.

**Just standardize on `ContiguousArray<UInt8>` and helper functions.** Rejected. The cursor model (`BytesReader` / `BytesWriter`) is the actual value-add — without it, every package re-implements offset threading and bounds checking. A "namespaced helpers on Array" design produces the same code we already have in Phase 1, just with different syntax.

**Adopt `swift-extras-bytes` (a hypothetical existing package).** No such package exists in the swift-server-community at sufficient maturity, so this isn't a real option. If the situation changes, RFC amendment.

**Make `Bytes` non-copyable.** Rejected. The buffer needs to be passed around as a result, stored in dictionaries (e.g., as a metric label cache), and round-tripped through APIs. Copying a `Bytes` is a CoW operation on the underlying `ContiguousArray`, so the cost is no worse than `[UInt8]`. Non-copyable would be ergonomically painful for marginal gain.

## Drawbacks

- **Yet Another Bytes Type.** The Swift ecosystem already has `[UInt8]`, `Data`, `ByteBuffer`, evolving `Span<UInt8>`, plus various ad-hoc wrappers in projects. Adding `Bytes` is +1. Mitigation: position it explicitly as the bare-swift in-package type, with first-class `[UInt8]` bridges. Don't try to displace ByteBuffer or Data; coexist with them.
- **CoW semantics under value-typed mutation.** A common pattern (`var w = BytesWriter(); w.writeByte(0); ...; let result = w.finish()`) relies on the underlying CoW being efficient. ContiguousArray's CoW is well-understood, but adopters writing inner loops should test for unintended copies. Document the gotcha.
- **`BytesReader` non-copyability** is unfamiliar to many Swift authors. Documentation needs to be explicit about why and how to model "two cursors over the same data" (the answer: hold two `BytesReader` values, each separately initialized from the same `Bytes`).
- **Once we ship `Bytes`, Phase 1 packages have a question to answer:** do they migrate from `[UInt8]` / `Sequence<UInt8>` to `Bytes`? This RFC doesn't force migration. swift-hex, swift-base64, swift-crc, etc. continue accepting `Sequence<UInt8>` (the bridges go via `Bytes(sequence)` if the user has `Bytes` in hand). Phase 2 packages adopt `Bytes` natively where it's the natural fit; Phase 1 packages stay as-is unless a v0.2 explicitly opts in.

## Unresolved questions

- **Hashing performance.** `Bytes: Hashable` synthesizes via `ContiguousArray<UInt8>: Hashable`, which hashes byte-by-byte. For label-keyed caches or hash-map use, that's fine for short Bytes (< ~128 bytes) but could be slow for larger payloads. Defer optimization until profiling proves it matters.
- **`BytesView` (non-owning, copyable view) as a separate type.** Possible v0.2 addition if profiling shows passing `Bytes` by value is a real cost in inner loops. v0.1 doesn't add one — `withUnsafeBufferPointer` is the borrow seam.
- **Endian-generic accessors.** `readInteger<T>(as: T.Type, endian: Endianness)` style would compress the API surface but add a generic-resolution cost. Defer.
- **Should `Bytes` be the storage type used internally by `BytesWriter`, or should `BytesWriter` own its own `ContiguousArray<UInt8>`?** Spec above uses `Bytes` directly so `finish()` is a move, not a copy. May revisit at impl time.

## Migration impact

- **No Phase 1 package needs to change.** swift-hex, swift-base64, swift-crc, swift-uuid, swift-varint, swift-xxhash, swift-jsonpointer, swift-dotenv, swift-ddsketch, swift-prometheus all continue to use `Sequence<UInt8>` / `[UInt8]` / `String` as their public surface.
- **Phase 2 packages adopt `Bytes` natively** where it's the right fit. swift-otlp-exporter takes `Bytes` for input and produces `Bytes` for output. swift-statsd writes into a `BytesWriter` and returns the finished `Bytes`. swift-hdrhistogram is `Bytes`-agnostic for its core API but exposes a serialization helper that writes a `Bytes` blob.
- **swift-bytes is a new package**, scaffolded via `bare-swift new` like every other package. Module name `Bytes`. Repo `bare-swift/swift-bytes`.
- **Future Phase 1 v0.2 amendments may opt in** to `Bytes` as an additional input type (e.g., swift-base64 v0.2 could add `Base64.encode(_ bytes: Bytes)` overloads), but RFC-0006 doesn't mandate any.

## Resolution

- **Accepted**
- Date: 2026-05-07
- Notes: swift-bytes becomes the first Phase 2 package to author (Tranche 2A). The v0.1 surface is the three types plus one error case described above; later RFC amendments add the deferred items (mutable slicing, `Span`/`RawSpan` bridges, endian-generic accessors). Phase 2's other packages (swift-otlp-exporter especially) target swift-bytes natively. Phase 1 packages do not migrate.

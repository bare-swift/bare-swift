# bare-swift launches with 10 Foundation-free packages

**Today:** 10 small Swift 6 packages — Foundation-free, Sendable-clean, typed-throws, dependency-light. All shipping at v0.1.0. All tested against canonical reference implementations. None compete with Apple-shipped libraries; every one fills a gap that pulled most Swift server projects into Foundation or NIO whether they wanted to be there or not.

This is the first of three planned phases. Phase 1 is the foundation; Phase 2 (observability — already specced) is next. Phase 3 will be config & serialization.

## What shipped

10 packages organized into three tiers by purpose:

### Encoding / hashing primitives

| Package | What it does | Translated from |
|---|---|---|
| [swift-hex](https://github.com/bare-swift/swift-hex) | Hex encoding/decoding (incl. constant-time) | Rust `hex` crate |
| [swift-base64](https://github.com/bare-swift/swift-base64) | Base64: standard / URL-safe, padded / unpadded, constant-time | Rust `base64` crate |
| [swift-crc](https://github.com/bare-swift/swift-crc) | CRC8 / 16 / 32 / 64 — generic engine + algorithm catalog | Rust `crc` crate |
| [swift-varint](https://github.com/bare-swift/swift-varint) | LEB128 (unsigned + signed) + ZigZag — protobuf / DWARF / WASM compatible | Rust `leb128` |
| [swift-xxhash](https://github.com/bare-swift/swift-xxhash) | xxHash (XXH32, XXH64, XXH3-64, XXH3-128) | Rust `twox-hash`, C `xxhash` |
| [swift-uuid](https://github.com/bare-swift/swift-uuid) | RFC 9562 UUIDs (v4, v7) and ULIDs | Rust `uuid`, `ulid-rs` |

### Format primitives

| Package | What it does | Translated from |
|---|---|---|
| [swift-jsonpointer](https://github.com/bare-swift/swift-jsonpointer) | RFC 6901 JSON Pointer (standard + URI fragment forms), JSON-tree agnostic | Rust `jsonptr` |

### Config & observability

| Package | What it does | Translated from |
|---|---|---|
| [swift-dotenv](https://github.com/bare-swift/swift-dotenv) | `.env` parser with `${VAR}` expansion | Rust `dotenvy` |
| [swift-ddsketch](https://github.com/bare-swift/swift-ddsketch) | Streaming quantile sketch with relative-error guarantees | Rust `sketches-ddsketch` (Datadog) |
| [swift-prometheus](https://github.com/bare-swift/swift-prometheus) | Prometheus metrics + `/metrics` text exposition (no swift-metrics dep) | Rust `prometheus-client` |

Browse the umbrella index: <https://bare-swift.github.io/bare-swift/>

## Why "Foundation-free"

Foundation is the obvious answer for many Swift problems. It's also a heavy dependency — bridging to Objective-C runtime types on Apple platforms, pulling in a lot of API surface, and historically the biggest reason Swift packages haven't worked cleanly in Embedded Swift or in minimal Linux containers.

bare-swift packages avoid Foundation **in their public API**. Internally, where pragmatic, packages use whatever's available (Glibc / Musl / Darwin for `log`, `pow` in numeric work). The contract is: a consumer can import a bare-swift package without inheriting `Data`, `URL`, `Date`, `NSObject` bridging, or any of Foundation's surface area in their own types.

Two practical consequences:

1. **You can pair bare-swift packages with whatever else you use.** A swift-nio service can use swift-prometheus without its types inheriting Foundation's dependency footprint. A future Embedded-Swift port has a smaller Foundation-removal task.
2. **Each package is small.** swift-prometheus is ~600 lines of Swift and has zero non-stdlib dependencies. swift-jsonpointer is ~250 lines. The pieces compose.

## Concrete examples

**Counting requests with swift-prometheus** (no swift-metrics involvement):

```swift
import Prometheus

let registry = MetricsRegistry()
let requests = try registry.counter(
    name: "http_requests_total",
    help: "Total HTTP requests served.",
    labels: ["method", "status"]
)
try requests.with(["method": "GET", "status": "200"]).inc()

// Render for /metrics
let body = registry.exposition()
```

**Quantile sketch with relative-error guarantee** (the sketch that doesn't get worse as your tail gets fatter):

```swift
import DDSketch

var sketch = DDSketch()             // 1% relative error, unbounded memory
for sample in latencies { sketch.add(sample) }
let p99 = try sketch.quantile(0.99)!   // guaranteed within ±1% of true
```

**Hashing with xxHash3** (matches `xxhsum` byte-for-byte):

```swift
import XXHash

let h = XXHash.xxh3_64("hello world".utf8)
// Or streaming:
var d = XXHash.Digest3_64()
d.update("hello, ".utf8)
d.update("world".utf8)
assert(d.finalize() == h)
```

**RFC 6901 JSON Pointer** (parse/format only — bring your own JSON tree):

```swift
import JSONPointer

let p = try JSONPointer("/foo/0")
print(p.tokens.map(\.value))    // ["foo", "0"]
print(p.tokens[1].arrayIndex!)  // 0

let q = try JSONPointer(uriFragment: "#/c%25d")
print(q.tokens[0].value)        // "c%d"
```

## How the packages relate

Six conventions hold across all 10 packages, codified as RFC-0001:

- Module names are PascalCase, no `Swift` prefix (`import UUID`, not `import SwiftUUID`).
- Apache-2.0 with LLVM exception. NOTICE crediting upstream Rust crate.
- Swift 6.0+ tools version, `.macOS(.v14)` floor (one exception: swift-prometheus uses `.macOS(.v15)` for `Synchronization.Atomic` — RFC-0003).
- Sendable by default; strict concurrency in CI.
- Single error enum per package; typed throws on the public surface.
- Foundation-free public API; pragmatic Foundation use in tests permitted.

The convention discipline is the value-add of having ten packages drop together rather than ten packages over a year. A reader who learned swift-hex's API shape can predict swift-base64's. A reader who saw swift-uuid's `parse → format → identity` round-trip test pattern recognizes the same shape in swift-jsonpointer.

## What's next

**Phase 2 anchor: observability wave** (RFC-0005). Three tranches over ~4 months:

- **2A:** swift-bytes (a span-like buffer; the prerequisite for binary-format work — RFC-0006), swift-hdrhistogram (HdrHistogram port).
- **2B:** swift-statsd (StatsD wire-format encoder), swift-otlp-exporter (OpenTelemetry protobuf exporter).
- **2C:** swift-prometheus 0.2 (Summary type, native histograms, OpenMetrics format), swift-prometheus-metrics (a swift-metrics adapter).

Phase 3 will likely be config & serialization (TOML, YAML, MessagePack, CBOR, JSONPath) but isn't decided yet — it'll get its own anchor RFC at Gate 2.

## How you can help

- **Try a package.** Even one. The fastest way for me to learn what's wrong is for someone to try shipping with one.
- **Open issues.** Bug reports, missing features, doc gaps — all useful.
- **Propose Phase 2 / 3 packages.** The ecosystem doc lists ~600 Rust crates worth porting; your favorite is probably in there.
- **Adopt the conventions.** If you've got an existing Foundation-free Swift package, the RFC-0001 conventions are public; you're welcome to align.

## Honest disclosures

- This is a **solo BDFL year-1 project**. Reviews and merges are best-effort. No SLA on issues yet.
- All 10 packages are **0.1.0**. Pre-1.0 SemVer applies — minor versions can break source compat. CHANGELOGs document all breaks.
- v1.0 happens **after at least one external user has adopted a package and survived an upgrade**. That's not a brag; it's a quality gate.
- These packages **don't compete with Apple-shipped types**. If `swift-foundation` ships an equivalent UUID type, swift-uuid stays valuable as a Foundation-free alternative — but you should use whichever fits.

## Links

- Umbrella + package index: <https://bare-swift.github.io/bare-swift/>
- All repos: <https://github.com/bare-swift>
- RFCs: <https://github.com/bare-swift/bare-swift/tree/main/rfcs>
- Roadmap: <https://github.com/bare-swift/bare-swift/blob/main/docs/superpowers/specs/2026-05-04-bare-swift-roadmap-design.md>

---

*Posted 2026-05-07. — bare-swift project lead.*

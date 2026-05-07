# Swift Forums post — Show & Tell or Server category

**Title:** bare-swift: 10 small Foundation-free Swift 6 packages, all v0.1.0 today

---

Hi all,

I'm announcing **bare-swift**, an ecosystem of small Foundation-free Swift 6 packages translated from well-known Rust crates. 10 packages shipped today as v0.1.0. None compete with Apple-shipped libraries; each fills a gap that historically pulled Swift server projects into Foundation.

**Umbrella + package index:** <https://bare-swift.github.io/bare-swift/>

## What's in Phase 1

Three tiers, 10 packages:

- **Encoding/hashing:** [swift-hex](https://github.com/bare-swift/swift-hex), [swift-base64](https://github.com/bare-swift/swift-base64), [swift-crc](https://github.com/bare-swift/swift-crc), [swift-varint](https://github.com/bare-swift/swift-varint), [swift-xxhash](https://github.com/bare-swift/swift-xxhash), [swift-uuid](https://github.com/bare-swift/swift-uuid)
- **Format:** [swift-jsonpointer](https://github.com/bare-swift/swift-jsonpointer)
- **Config / observability:** [swift-dotenv](https://github.com/bare-swift/swift-dotenv), [swift-ddsketch](https://github.com/bare-swift/swift-ddsketch), [swift-prometheus](https://github.com/bare-swift/swift-prometheus)

## What "Foundation-free" means here

Public APIs don't expose `Data`, `URL`, `Date`, `NSObject` bridging, or any other Foundation types. Internal use is permitted where pragmatic. The contract is: importing a bare-swift package doesn't inherit Foundation's surface area into your own types.

## Conventions

Six conventions hold across all 10 packages (codified as RFC-0001):

- PascalCase module names, no `Swift` prefix (`import UUID`, not `import SwiftUUID`).
- Apache-2.0 + LLVM exception. NOTICE crediting upstream Rust crate.
- Swift 6.0+; `.macOS(.v14)` floor with one exception (swift-prometheus uses `.macOS(.v15)` for `Synchronization.Atomic` — explained in RFC-0003).
- Sendable by default; strict concurrency in CI.
- Single error enum per package; Swift 6 typed throws.
- Foundation-free public API; tests may use Foundation pragmatically.

## Notable bits

**swift-prometheus** is standalone — no `swift-metrics` dependency. The tradeoff: it doesn't auto-bridge into the swift-metrics ecosystem. Phase 2 will ship a separate `swift-prometheus-metrics` adapter for that.

**swift-ddsketch** has the relative-error quantile guarantee that's actually useful when your latency distribution is fat-tailed. Cross-validated against Datadog's Python `ddsketch` 3.0.1: bit-exact `count`/`min`/`max`/`sum` and within-α quantiles.

**swift-xxhash** is wire-compatible with the C `xxhsum` tool — golden vectors generated from `xxhsum --tag` are baked into the test suite.

## Concrete code

```swift
import Prometheus

let registry = MetricsRegistry()
let requests = try registry.counter(
    name: "http_requests_total",
    help: "Total HTTP requests.",
    labels: ["method", "status"]
)
try requests.with(["method": "GET", "status": "200"]).inc()

let body = registry.exposition()    // canonical /metrics output
```

```swift
import DDSketch

var sketch = DDSketch()
for sample in latencies { sketch.add(sample) }
let p99 = try sketch.quantile(0.99)!  // ±1% relative error
```

## What's next

Phase 2 (RFC-0005) anchors on the observability wave: swift-bytes (RFC-0006-defined), swift-hdrhistogram, swift-statsd, swift-otlp-exporter, swift-prometheus 0.2, and a swift-prometheus-metrics adapter. ~4 months, three tranches.

Phase 3 (probably config & serialization — TOML, YAML, MessagePack, CBOR, JSONPath) gets its own anchor RFC at Gate 2.

## What I'd love feedback on

- **Adoption:** if you try one of these in a real service, please open an issue (bug or feature request). Even a "this didn't fit my use case because…" is gold.
- **API consistency:** if something feels surprising relative to the rest of the ecosystem, that's the thing I most want to hear about.
- **Phase 2 scope:** RFC-0005 is open for amendment-by-PR if there's a Phase 2 package you think we shouldn't ship, or one we should that's not on the list.
- **swift-metrics integration:** swift-prometheus is intentionally standalone; the adapter ships in Phase 2 (Tranche 2C). If that ordering is wrong for your use case, please weigh in.

## Disclosures

- Solo BDFL project (Year 1). Best-effort review SLAs.
- All packages are 0.1.0. Pre-1.0 SemVer; minor versions can break.
- v1.0 only after one external user adopts and survives an upgrade.

## Links

- Umbrella: <https://bare-swift.github.io/bare-swift/>
- Org: <https://github.com/bare-swift>
- RFCs: <https://github.com/bare-swift/bare-swift/tree/main/rfcs>
- Roadmap: <https://github.com/bare-swift/bare-swift/blob/main/docs/superpowers/specs/2026-05-04-bare-swift-roadmap-design.md>

Thanks for reading; happy to answer questions in this thread.

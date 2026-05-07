# HN / Lobsters submission

## Title (HN, ≤80 chars)

bare-swift: 10 small Foundation-free Swift 6 packages, all v0.1.0

## URL

https://bare-swift.github.io/bare-swift/

## Optional first comment / Lobsters story text

Submitting bare-swift, which I've been building over the last few weeks. It's an ecosystem of 10 small Swift 6 packages — Foundation-free, Sendable, typed throws — translated from well-known Rust crates. All ten shipped today as v0.1.0.

The packages cover encoding (hex, base64, crc, varint, xxhash, uuid), format (jsonpointer), and config/observability (dotenv, ddsketch, prometheus). They're individually useful and they compose; the underlying thesis is that a Foundation-free Swift package layer is missing from the ecosystem and that the way to build it is to translate the Rust ecosystem's most-depended-upon crates one at a time, with a consistent set of conventions.

A few notable bits:

- **swift-ddsketch** has the relative-error quantile guarantee. Cross-validated against Datadog's Python `ddsketch` 3.0.1.
- **swift-prometheus** is standalone — no `swift-metrics` dependency. (A swift-metrics adapter ships in Phase 2.)
- **swift-xxhash** is byte-exact wire-compatible with the C `xxhsum` tool.
- **swift-uuid** ships v4 + v7 plus ULIDs. RFC 9562 conformant.

Conventions are codified as RFCs (RFC-0001 through RFC-0006 already accepted; the public RFC ledger is at <https://github.com/bare-swift/bare-swift/tree/main/rfcs>). Phase 2 anchors on the observability wave (HdrHistogram, OTLP exporter, StatsD, expanded Prometheus) and is fully specced in RFC-0005.

Honest disclosures: solo project, Year 1, best-effort review SLAs, pre-1.0 packages may break, v1.0 only after at least one external adopter.

Open to feedback, especially if you try a package and find an API rough edge.

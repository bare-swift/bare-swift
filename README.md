# bare-swift

A public Swift Package ecosystem of small, independently-useful libraries that fill concrete gaps in today's Swift-server world. Each package corresponds to a Rust crate (or family) whose design we translate to Swift idioms.

## What this is

- **Composable.** Every package is small, focused, and depends on as little as possible.
- **Sendable-clean.** Strict concurrency on. Public APIs are `Sendable` by default.
- **Foundation-free APIs.** Use packages without pulling in Foundation. (Internal use is fine.)
- **Documented.** DocC bundle for every package.
- **Test-parity.** Every package passes the source Rust crate's test vectors.

## What this is NOT

- A replacement for swift-nio, swift-crypto, swift-collections, swift-system, swift-async-algorithms, swift-syntax, swift-argument-parser, swift-log, swift-metrics, or swift-distributed-tracing. We *complement* those, not compete.
- A pure-Swift reimplementation of crypto, TLS, compression, SQLite, or ICU. Where the C library is the standard, we wrap it.
- A monolithic framework. Each library stands alone.

## Packages

See [`packages/index.json`](./packages/index.json) for the full list. The website at https://bare-swift.github.io renders a friendlier version.

## RFCs

Cross-package design decisions live in [`rfcs/`](./rfcs/). The two foundational RFCs:

- [RFC-0001 — API Conventions](./rfcs/0001-api-conventions.md)
- [RFC-0002 — Test Parity Policy](./rfcs/0002-test-parity-policy.md)

## Roadmap

The full ecosystem map lives at [`docs/ecosystem.md`](./docs/ecosystem.md). The active phasing plan is in the project's spec directory.

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md). All participation is governed by the [Code of Conduct](./CODE_OF_CONDUCT.md).

## Security

Report vulnerabilities per [SECURITY.md](./SECURITY.md).

## License

Apache 2.0 with LLVM exception. See [LICENSE](./LICENSE).

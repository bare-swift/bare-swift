# RFC-0001 — API Conventions

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-04 |
| Resolution | Accepted 2026-05-04 — codifies the conventions section of the lift-off roadmap. |

## Summary

This RFC codifies the public API conventions every bare-swift package must follow. It's the source of truth referenced by `CONTRIBUTING.md`, the scaffold tool, and CI lints.

## Problem

Phase 1 will produce 10 packages over 4–6 months. Without enforced conventions decided up front, those packages drift in style, error model, threading model, and dependency footprint. Drift means refactoring 30 packages later. Better to decide once.

## Proposal

### Org and naming

- GitHub org: `bare-swift`.
- One repo per package. Repo name: `swift-<crate>` (e.g., `swift-uuid`, `swift-toml`).
- Module name: PascalCase, no `Swift` prefix. `import UUID`, not `import SwiftUUID`.
- API names follow the [Swift API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/), not Rust naming.

### License

- Apache-2.0 with LLVM exception. SPDX header at the top of every source file:

  ```swift
  // SPDX-License-Identifier: Apache-2.0 WITH LLVM-exception
  // Copyright (c) 2026 The bare-swift Project Authors.
  ```

- A `NOTICE` file in every repo crediting the source Rust crate, original author, and the crate's license.

### Swift target and platforms

- Minimum Swift 6.0 (`swift-tools-version: 6.0` in Package.swift).
- macOS 14+ and Linux (Ubuntu 22.04 + 24.04) are required.
- Windows is best-effort. If a package excludes Windows, document why in its README.
- Embedded Swift is out of scope for v1; flag as future work where applicable.

### Concurrency

- **Sendable-clean by default.** Every public type either is `Sendable` or has a doc comment explaining why not.
- `@unchecked Sendable` requires an inline comment justifying the unchecked claim ("the underlying buffer is immutable after init"). Reviewer will reject without one.
- Strict concurrency mode in CI (`-strict-concurrency=complete`).

### Errors

- Each package defines a single public error enum (e.g., `UUIDError`, `TOMLError`).
- Public throwing functions use Swift 6 typed throws: `func parse(_ input: String) throws(UUIDError) -> UUID`.
- `throws` (untyped) is reserved for `internal` and `private` code.

### Async

- I/O packages expose async APIs.
- Pure-logic packages (encoders, hashes, parsers) stay sync. Don't add `async` for symmetry.

### Foundation

- Public APIs do not expose Foundation types (`Data`, `URL`, `Date`, `NSNumber`, etc.).
- Use `[UInt8]`, `ContiguousArray<UInt8>`, or a span-like type for bytes; use `String` for text; define purpose-specific types for everything else.
- Internal use of Foundation is permitted where pragmatic (`FileManager`, `JSONDecoder` for tests). Mark Foundation imports `@_implementationOnly` where the toolchain supports it.
- Exception: a package whose stated purpose *is* to extend Foundation (we don't ship any in Phase 1).

### Repo skeleton

Every package repo has the layout produced by `bare-swift new`:

```
swift-<name>/
  Package.swift
  Sources/<Module>/...
  Tests/<Module>Tests/...
  Tests/Vectors/                 (test fixtures from source Rust crate)
  Documentation.docc/
  README.md                      (tagline, install, ≤30-line example)
  CHANGELOG.md                   (keepachangelog format)
  NOTICE                         (source crate credit)
  LICENSE                        (Apache-2.0 + LLVM exception)
  MAINTAINERS.md
  SECURITY.md
  .gitignore
  .github/workflows/ci.yml       (consumes shared workflow via uses:)
  .github/ISSUE_TEMPLATE/
  .github/PULL_REQUEST_TEMPLATE.md
```

### Versioning

- Semantic versioning.
- Pre-1.0 (0.x): API can break between minor versions. Document breaks in CHANGELOG.
- 1.0+: source-stable. Run `swift package diagnose-api-breaking-changes` against the previous tag in CI.
- v1.0 only after **at least one external user** (not the org) has adopted the package and survived an upgrade.

### Quality bar before v0.1

A package does not ship publicly until:

- All public API matches this RFC.
- Test vectors from the source Rust crate are extracted and passing.
- DocC bundle exists with a one-page overview and at least one full code example per public type.
- README is complete (tagline, install, ≤30-line working example, link to DocC, link to source Rust crate).
- CI is green on macOS + Linux matrix.
- CHANGELOG has a v0.1 entry.

In year 1 with solo BDFL, "review" means self-review against this checklist before tagging.

## Alternatives considered

**Allow Foundation types in public APIs.** Rejected: closes the door on Foundation-free environments (Embedded Swift in particular). Short-term ergonomic win, long-term composability loss.

**Untyped `throws Error` everywhere.** Rejected: typed throws are why we waited for Swift 6. Throwing typed errors in libraries lets consumers `catch` exhaustively without `default`.

**Allow `@unchecked Sendable` without justification comments.** Rejected: this convention exists because reviewers can't verify the claim without context. The comment is the proof.

**One mega-repo with all packages as members of a workspace.** Rejected: independent versioning, independent release, independent issue tracking. The cost (small extra `swift package edit` ceremony) is worth it for adopter clarity.

## Drawbacks

- "No Foundation in public APIs" makes some package APIs feel less Swift-native to existing Foundation-using teams. They have to convert at boundaries (`Data(_:)`, `Array(data)`).
- Typed throws still don't compose perfectly across functions that combine errors from multiple sources. Some packages will need a sum-type error to wrap upstream errors, which is verbose.
- "v1.0 only after one external user" delays the 1.0 milestone artificially. Some packages may sit at 0.9 indefinitely.

## Unresolved questions

- What's the policy for packages that legitimately need to expose `URL`? (None in Phase 1 do; revisit when the question arises.)
- How do we handle packages that wrap a C library (the doc says "wrap, don't reimplement" for crypto/TLS/etc.)? Probably a separate RFC when the first such package is proposed.

## Migration impact

This is the founding convention RFC. No existing packages to migrate.

## Resolution

**Accepted 2026-05-04.** This RFC governs all packages from Phase 0 onward. Future amendments are themselves RFCs.

# RFC-0002 — Test Parity Policy

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-04 |
| Resolution | Accepted 2026-05-04 — required for every package shipped from Phase 0 onward. |

## Summary

Every bare-swift package extracts test vectors (input/output pairs, parser fuzz corpora, regression cases) from its source Rust crate and runs them inside Swift tests. This RFC defines what "test parity" means, when it can be relaxed, and how vectors are extracted.

## Problem

Translating an algorithm from Rust to Swift is the easy part. Knowing whether the translation is *correct* — across edge cases, malformed inputs, byte-level outputs, locale quirks — is the hard part. Rust crates have years of accumulated test cases that encode hard-won knowledge. Re-deriving those by inspection is slow and risks blind spots.

The same vectors must run in Swift, identically passing.

## Proposal

### What "test parity" means

For every public function in a bare-swift package whose Rust counterpart has tests:

- The bare-swift tests include a vector-driven test that runs the same inputs through the Swift implementation and asserts the same outputs.
- "Same outputs" means byte-equal for binary, character-equal for text, structurally-equal for typed values.

### Where vectors live

Inside each package: `Tests/Vectors/` — a directory of files copied verbatim from the source Rust crate's test fixtures. SwiftPM declares this as a test target resource:

```swift
.testTarget(
    name: "<Module>Tests",
    dependencies: ["<Module>"],
    resources: [.copy("../Vectors")]
)
```

Files keep their original Rust-side relative paths under `Tests/Vectors/` for traceability. Each top-level subdirectory under `Vectors/` is named after the source file or test group it came from.

### How vectors are extracted

The `bare-swift extract-vectors` CLI subcommand handles this. Each package has a `Tests/Vectors/MANIFEST.txt`:

```
# One path or glob per line. Resolved relative to the source crate root.
# Lines starting with # are comments.
tests/data/*.txt
tests/fixtures/encode/*.bin
benches/data/*.json
```

Running `bare-swift extract-vectors --crate <path-to-rust-crate> --manifest Tests/Vectors/MANIFEST.txt --output Tests/Vectors/` copies the listed files into `Tests/Vectors/` preserving their relative paths.

The MANIFEST is committed; the extracted files are also committed (so users don't need the Rust source to run tests).

### Running vectors in Swift

Each package writes a Swift test that loads the vectors and runs them. A typical pattern:

```swift
@Test func passesAllUpstreamVectors() throws {
    let vectorsURL = Bundle.module.url(forResource: "Vectors", withExtension: nil)!
    let dataDir = vectorsURL.appendingPathComponent("tests/data")
    for url in try FileManager.default.contentsOfDirectory(at: dataDir, includingPropertiesForKeys: nil) {
        let input = try String(contentsOf: url, encoding: .utf8)
        let expected = try String(contentsOf: url.deletingPathExtension().appendingPathExtension("expected"), encoding: .utf8)
        #expect(MyParser.parse(input) == expected)
    }
}
```

(Each package adapts the loader to its vector format.)

### When parity can be relaxed

Documented exceptions only, in a `Tests/Vectors/EXCEPTIONS.md` file:

- **Behavior intentionally differs.** E.g., a Rust crate accepts non-UTF-8 sequences that we reject. Document the input, the Rust behavior, the Swift behavior, and the rationale.
- **Vector depends on Rust-only infrastructure.** E.g., relies on a Rust-specific test harness or cargo feature flag. Document and skip; ideally mark as "we should add a Swift-equivalent test."
- **Vector tests deprecated/removed Rust behavior.** Skip and document.

A package without `EXCEPTIONS.md` is asserting "all extracted vectors pass."

### Refresh cadence

- At scaffold time, vectors are pulled from a specific upstream commit. The MANIFEST records the commit hash.
- When the source crate releases a new version, the package maintainer re-runs `extract-vectors` against the new commit, reviews any vector changes, and either updates the Swift implementation or documents an EXCEPTION.

### CI

- The vectors test runs in standard CI; failure blocks merge.
- A separate scheduled workflow (monthly) checks whether the source crate has advanced beyond the recorded commit and opens an issue if so.

## Alternatives considered

**Hand-translate vectors into Swift literals in the test files.** Rejected: defeats the point. Translation introduces bugs; refresh becomes manual; reviewers can't easily diff against upstream.

**Reference vectors from a separate `bare-swift-vectors` repo.** Rejected: per-package vectors mean per-package reproducibility. Pulling vectors from another repo at test time adds CI flakiness and breaks offline builds.

**Run Rust tests via FFI from Swift.** Rejected: enormous toolchain complexity for a benefit (running upstream tests verbatim) that copying-the-vectors achieves more cleanly.

## Drawbacks

- Repos grow larger by however many vectors the crate ships. Most are small (text/binary fixtures); some crates have multi-MB corpora. Per-package, the maintainer can prune large fixtures with EXCEPTIONS.
- Refresh isn't automated yet — the monthly CI check just opens an issue. A future iteration could auto-PR a vector refresh.
- Some Rust crates write tests using generators (`proptest`, `quickcheck`). Generator-driven tests don't extract; the Swift package writes its own equivalents using `swift-testing` or hand-rolled property tests.

## Unresolved questions

- For crates with multiple "phases" of test vectors (e.g., Unicode tables versioned to a specific Unicode release), which version do we track? Per-package decision recorded in the MANIFEST.
- For vectors larger than 5 MB, do we use Git LFS? Defer until a real package needs it.

## Migration impact

This is the founding test policy. No existing packages to migrate.

## Resolution

**Accepted 2026-05-04.** Required for all packages from Phase 0 onward. The scaffold tool and `extract-vectors` subcommand are designed to make compliance frictionless.

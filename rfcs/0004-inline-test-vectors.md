# RFC-0004 — Inline Swift test vectors as the default extraction mechanism

| Field | Value |
| --- | --- |
| Status | Draft |
| Author | bare-swift project lead |
| Created | 2026-05-07 |
| Resolution | _(filled at merge)_ |

## Summary

Amend [RFC-0002](./0002-test-parity-policy.md) to make **inline Swift literals** the default form of test vectors, with file-based vectors (`Tests/Vectors/` resources via `Bundle.module`) as the exception. Every Phase 1 package shipped this way; this RFC matches policy to observed reality. RFC-0002's MANIFEST mechanism remains valid for crates that publish data fixtures, but it is no longer the default expectation.

## Problem

RFC-0002 §"Where vectors live" prescribes:

- A `Tests/Vectors/` directory of files copied verbatim from the Rust crate.
- A `Tests/Vectors/MANIFEST.txt` listing source paths in the upstream crate.
- A `bare-swift extract-vectors` CLI that copies the listed files.
- Swift tests that load via `Bundle.module.url(forResource: "Vectors", ...)` and `FileManager.default.contentsOfDirectory(...)`.

RFC-0002 also explicitly **rejected** "hand-translate vectors into Swift literals" as an alternative. The rejection rationale: "translation introduces bugs; refresh becomes manual; reviewers can't easily diff against upstream."

Phase 1 result: every one of the 10 packages used inline Swift literals. Zero packages used the file-based MANIFEST mechanism. The reasons cluster around three observed patterns:

1. **Most upstream crates have inline `#[test]` cases, not data fixtures.** `leb128`, `dotenvy`, `jsonptr`, `crc`, `hex`, `twox-hash`, `uuid`, `prometheus-client`, `sketches-ddsketch` — none ship dedicated data files. Their tests are Rust source code (typically with hard-coded inputs). Extracting "verbatim" is impossible because there's no extractable file.

2. **Foundation-free runtime loaders are awkward.** RFC-0002's example loader uses `Bundle.module.url(...)`, `FileManager.default.contentsOfDirectory(...)`, and `String(contentsOf:)` — all Foundation. RFC-0001 permits Foundation in tests, but every package's main code path is Foundation-free, and the test code still has to handle the Foundation-vs-not split. Inline literals avoid the split.

3. **Specs frequently come from RFCs or documents, not code.** RFC 6901 (JSON Pointer) tables, RFC 6962 example outputs, the Prometheus exposition format spec — these are document sources. The source-of-truth is the RFC text, not a Rust crate's test files. Extracting from Rust to a "verbatim file" misrepresents the lineage.

The right reading of RFC-0002's intent is: **"every test the Rust crate runs, the Swift package also runs."** That intent is satisfied by inline literals. The mechanism RFC-0002 specified turned out to fit a minority of cases.

## Proposal

### Default: inline Swift literals

Test vectors live as Swift literals inside the test files, structured as one of:

- A static array of test rows (input → expected) in the test struct, iterated by a `@Test` function.
- Per-case `@Test` functions when each case has distinct setup or rationale.

Whichever shape a package chooses, the vectors stay co-located with the assertions that consume them. No runtime loading, no Foundation file IO, no `Bundle.module` lookups in the default case.

### `Tests/Vectors/EXCEPTIONS.md` is mandatory

Every package keeps `Tests/Vectors/EXCEPTIONS.md` with three required sections:

1. **Source `<crate>` — what's translated and how.** One paragraph per upstream source explaining whether vectors come from inline `#[test]` cases, data fixtures, generators (proptest / quickcheck), or specs (RFC documents, etc.).

2. **Out of scope for v0.1 (no Swift counterpart).** Bullet list of upstream test categories that have no Swift counterpart, with one-line reasons. Tested-against-features-we-don't-implement is the most common bullet.

3. **Refresh.** How to re-derive vectors when the upstream advances a version (or when a new spec revision lands). Specifies the exact CLI invocation, Python script, or table reference used at the time of v0.1.

### File-based vectors remain valid for genuine data fixtures

When an upstream crate genuinely ships data files (e.g., a Unicode-data table, a TLS test corpus, a fuzzer-generated regression set), the package may use the original RFC-0002 mechanism: `Tests/Vectors/<files>`, MANIFEST, `Bundle.module` loader. The package's `EXCEPTIONS.md` notes "vectors are file-based per RFC-0002" rather than "vectors are inline per RFC-0004."

Either pattern satisfies the test-parity intent. The choice is per-package.

### External generation is permitted and encouraged when it's the source of truth

When the canonical reference is a non-Rust implementation (e.g., the C `xxhsum` tool for swift-xxhash; the Python `ddsketch` package for swift-ddsketch), generating vectors from that reference is preferred over hand-deriving. The `EXCEPTIONS.md` "Refresh" section names the exact generator and version pinned at v0.1.

This is what swift-xxhash and swift-ddsketch did at v0.1; both packages used a Python venv with the canonical reference to print vectors at plan-execution time, then baked the printed values inline.

### Property-driven tests stay required

RFC-0002's note that property tests "don't extract; the Swift package writes its own equivalents using `swift-testing` or hand-rolled property tests" is unchanged. Phase 1 uniformly used a deterministic LCG (no Foundation) to drive property tests; RFC-0004 names this as the canonical pattern.

A per-package implementation of an LCG is acceptable; we are not yet promoting it to a shared utility. Phase 2 may revisit if the duplication grows tiresome.

## Alternatives considered

**Keep RFC-0002 as-is and treat the 10 inline-vector packages as exceptions.** Rejected: 10/10 packages cannot all be exceptions to a policy. Either the policy is mis-targeted, or it's de facto repealed. Better to amend.

**Supersede RFC-0002 entirely with an inline-only rule.** Rejected: file-based vectors are the right mechanism for some upstreams (Unicode tables, IETF corpora). A package that wraps `unicode-segmentation` legitimately needs the data files. Keep the option.

**Require both forms — inline for assertions, files for upstream parity.** Rejected: doubles the maintenance burden for marginal gain. Inline literals already trace to upstream via `EXCEPTIONS.md`.

**Auto-generate inline vectors from the upstream Rust crate via a build-time script.** Rejected for Phase 2: tooling cost outweighs benefit. The vectors are stable; once baked, they don't drift. Refresh is a manual step at version-bump time, called out in `EXCEPTIONS.md`.

## Drawbacks

- **Refreshing inline vectors is manual.** When upstream `dotenvy` advances a version, the package maintainer re-reads the new tests and updates the inline literals. This is the cost of avoiding the file-extraction machinery.

- **No automated drift detector.** RFC-0002's monthly CI workflow that compares against the recorded upstream commit is mooted for inline-vector packages. We accept this tradeoff: the bare-swift packages are pinned snapshots; we re-baseline at version bumps.

- **Reviewers cannot mechanically diff against upstream.** A reviewer can't run `diff <inline> <upstream test file>`. They have to spot-check by reading both. The `EXCEPTIONS.md` "Refresh" recipe is the contract: anyone can re-run the recipe and check that the inline values match.

## Unresolved questions

- For a future package whose upstream is so heavily fixture-driven that file-based vectors are the only sensible choice (Unicode-segmentation candidate), do we keep RFC-0002's `extract-vectors` CLI, or maintain a one-off script in that package's `scripts/` directory? Defer until that package is actually proposed.

- Vector freshness: should the umbrella site display "vectors derived from `dotenvy 0.15.7`" alongside each package? Probably yes, but it's a `packages/index.json` schema change, tracked separately.

## Migration impact

This RFC changes wording, not code. The 10 Phase 1 packages already conform.

- `RFC-0002` is amended in spirit, not deleted. A short note appended to RFC-0002 should point readers to RFC-0004 for the practical default.
- `bare-swift new`'s scaffold continues to create `Tests/Vectors/EXCEPTIONS.md` with a stub that references both RFCs.
- The `bare-swift extract-vectors` CLI subcommand stays available but is no longer the default invocation in `CONTRIBUTING.md`.

## Resolution

_(Filled by the project lead at merge time.)_

- **Accepted** / **Rejected** / **Withdrawn**
- Date:
- Notes:

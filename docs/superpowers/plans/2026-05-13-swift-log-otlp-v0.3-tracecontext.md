# swift-log-otlp v0.3.0 — TraceContext convenience init (Phase 14 Tranche 14C) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a convenience initializer on `OTLP.LogRecord` accepting `OTLP.TraceContext` that fills the cross-signal correlation fields (`traceID`, `spanID`, `flags`), closing Phase 14 per RFC-0019.

**Architecture:** Additive-only minor version bump. Add `swift-tracing-otlp` 0.3.0 as a new dependency (it owns the `OTLP.TraceContext` type added in 14B). New file `TraceContextIntegration.swift` follows the same shape as the existing `TimeIntegration.swift` — extension on `OTLP.LogRecord` with one new initializer that delegates to the canonical init. No changes to existing types, fields, or wire format. Per the OTLP spec, the low 8 bits of `LogRecord.flags` carry the W3C `traceFlags` byte, so `flags = UInt32(traceContext.traceFlags)`.

**Tech Stack:** Swift 6.0, swift-bytes, swift-varint, swift-otlp-exporter, swift-time, **swift-tracing-otlp (new, 0.3.0)**, swift-testing.

---

## File Structure

- **Modify:** `Package.swift` — add `swift-tracing-otlp` 0.3.0 dependency and product.
- **Create:** `Sources/LogOTLP/TraceContextIntegration.swift` — new convenience initializer (~25 LOC).
- **Create:** `Tests/LogOTLPTests/TraceContextIntegrationTests.swift` — `@Suite` covering 6 behaviors (~80 LOC).
- **Modify:** `CHANGELOG.md` — add `[0.3.0] - 2026-05-13` section.
- **Modify:** `README.md` — bump install snippet, add usage block.
- **Modify:** `Sources/LogOTLP/Documentation.docc/LogOTLP.md` — short topic blurb (no DocC cross-package symbol link — see `feedback_docc_cross_package`).
- **Modify (umbrella):** `/Users/satishbabariya/Desktop/swift-libs/bare-swift/packages/index.json` — bump `swift-log-otlp` 0.2.0 → 0.3.0.

---

## Task 1: Add swift-tracing-otlp dependency

**Files:**
- Modify: `/Users/satishbabariya/Desktop/swift-libs/swift-log-otlp/Package.swift`

- [ ] **Step 1: Add the package dependency and product**

In `Package.swift`, add `swift-tracing-otlp` to the `dependencies` array and `TracingOTLP` to the target's `dependencies` array.

Change the `dependencies` block from:

```swift
    dependencies: [
        .package(url: "https://github.com/swiftlang/swift-docc-plugin.git", from: "1.4.0"),
        .package(url: "https://github.com/bare-swift/swift-bytes.git", from: "0.1.0"),
        .package(url: "https://github.com/bare-swift/swift-varint.git", from: "0.1.0"),
        .package(url: "https://github.com/bare-swift/swift-otlp-exporter.git", from: "0.1.0"),
        .package(url: "https://github.com/bare-swift/swift-time.git", from: "0.1.0")
    ],
```

to:

```swift
    dependencies: [
        .package(url: "https://github.com/swiftlang/swift-docc-plugin.git", from: "1.4.0"),
        .package(url: "https://github.com/bare-swift/swift-bytes.git", from: "0.1.0"),
        .package(url: "https://github.com/bare-swift/swift-varint.git", from: "0.1.0"),
        .package(url: "https://github.com/bare-swift/swift-otlp-exporter.git", from: "0.1.0"),
        .package(url: "https://github.com/bare-swift/swift-time.git", from: "0.1.0"),
        .package(url: "https://github.com/bare-swift/swift-tracing-otlp.git", from: "0.3.0")
    ],
```

And change the target's `dependencies` array from:

```swift
            dependencies: [
                .product(name: "Bytes", package: "swift-bytes"),
                .product(name: "Varint", package: "swift-varint"),
                .product(name: "OTLPExporter", package: "swift-otlp-exporter"),
                .product(name: "Time", package: "swift-time")
            ]
```

to:

```swift
            dependencies: [
                .product(name: "Bytes", package: "swift-bytes"),
                .product(name: "Varint", package: "swift-varint"),
                .product(name: "OTLPExporter", package: "swift-otlp-exporter"),
                .product(name: "Time", package: "swift-time"),
                .product(name: "TracingOTLP", package: "swift-tracing-otlp")
            ]
```

- [ ] **Step 2: Verify resolution + build**

Run from `/Users/satishbabariya/Desktop/swift-libs/swift-log-otlp`:

```bash
swift build
```

Expected: `Build complete!` Should pull in `swift-tracing-otlp` and `swift-hex` from the registry. The existing 4 source files compile unchanged.

- [ ] **Step 3: Run the existing test suite as a baseline**

```bash
swift test 2>&1 | tail -3
```

Expected: All existing v0.2 tests still pass. Note the count (it stays the same after this task).

- [ ] **Step 4: Commit**

```bash
git add Package.swift
git commit -m "build: add swift-tracing-otlp 0.3.0 dependency

For the upcoming OTLP.LogRecord(traceContext:) convenience init that
fills cross-signal correlation fields (Phase 14 Tranche 14C)."
```

---

## Task 2: Write the failing test for the convenience initializer

**Files:**
- Create: `/Users/satishbabariya/Desktop/swift-libs/swift-log-otlp/Tests/LogOTLPTests/TraceContextIntegrationTests.swift`

- [ ] **Step 1: Create the test file with the first failing test**

Create the file with this exact content:

```swift
// SPDX-License-Identifier: Apache-2.0 WITH LLVM-exception
// Copyright (c) 2026 The bare-swift Project Authors.

import Bytes
import OTLPExporter
import Testing
import TracingOTLP
@testable import LogOTLP

@Suite("OTLP.LogRecord traceContext convenience init")
struct TraceContextIntegrationTests {
    private static let canonicalTraceID = Bytes([
        0x4b, 0xf9, 0x2f, 0x35, 0x77, 0xb3, 0x4d, 0xa6,
        0xa3, 0xce, 0x92, 0x9d, 0x0e, 0x0e, 0x47, 0x36
    ])
    private static let canonicalSpanID = Bytes([
        0x00, 0xf0, 0x67, 0xaa, 0x0b, 0xa9, 0x02, 0xb7
    ])

    @Test("traceContext init fills traceID/spanID/flags from the context")
    func fillsCorrelationFields() {
        let ctx = OTLP.TraceContext(
            traceID: Self.canonicalTraceID,
            spanID: Self.canonicalSpanID,
            traceFlags: 0x01
        )
        let record = OTLP.LogRecord(
            timeUnixNano: 1_700_000_000_000_000_000,
            severityNumber: .info,
            body: .string("hello"),
            traceContext: ctx
        )
        #expect(record.traceID == Self.canonicalTraceID)
        #expect(record.spanID == Self.canonicalSpanID)
        #expect(record.flags == 0x01)
        #expect(record.timeUnixNano == 1_700_000_000_000_000_000)
        #expect(record.severityNumber == .info)
        #expect(record.body == .string("hello"))
    }
}
```

- [ ] **Step 2: Run it to verify it fails**

```bash
swift test --filter "TraceContextIntegrationTests" 2>&1 | tail -10
```

Expected: Compile error. The error will be along the lines of `error: extra argument 'traceContext' in call` or `error: incorrect argument label in call (have ..., expected ...)`. We have not added the initializer yet.

- [ ] **Step 3: Commit the failing test**

```bash
git add Tests/LogOTLPTests/TraceContextIntegrationTests.swift
git commit -m "test: failing test for OTLP.LogRecord(traceContext:) init"
```

---

## Task 3: Implement the convenience initializer

**Files:**
- Create: `/Users/satishbabariya/Desktop/swift-libs/swift-log-otlp/Sources/LogOTLP/TraceContextIntegration.swift`

- [ ] **Step 1: Write the new source file**

Create the file with this exact content:

```swift
// SPDX-License-Identifier: Apache-2.0 WITH LLVM-exception
// Copyright (c) 2026 The bare-swift Project Authors.

import Bytes
import OTLPExporter
import TracingOTLP

/// swift-tracing-otlp integration for OTLP log records.
///
/// `OTLP.TraceContext` (defined in swift-tracing-otlp) is the W3C Trace
/// Context propagation value used to thread trace IDs across HTTP
/// boundaries. OTLP cross-signal correlation requires the same trace ID,
/// span ID, and trace-flags byte to appear on each `LogRecord` emitted
/// in the scope of a span. This initializer fills those fields from a
/// single `TraceContext` value to avoid repetitive plumbing at call
/// sites.
///
/// Per OTLP convention, the low 8 bits of `LogRecord.flags` carry the
/// W3C `traceFlags` byte. The high 24 bits are reserved; this init
/// leaves them zero.
extension OTLP.LogRecord {
    /// Convenience initializer that fills `traceID`, `spanID`, and
    /// `flags` from an `OTLP.TraceContext`.
    public init(
        timeUnixNano: UInt64 = 0,
        observedTimeUnixNano: UInt64 = 0,
        severityNumber: OTLP.SeverityNumber = .unspecified,
        severityText: String = "",
        body: OTLP.AnyValue? = nil,
        attributes: [OTLP.KeyValue] = [],
        droppedAttributesCount: UInt32 = 0,
        traceContext: OTLP.TraceContext,
        eventName: String = ""
    ) {
        self.init(
            timeUnixNano: timeUnixNano,
            observedTimeUnixNano: observedTimeUnixNano,
            severityNumber: severityNumber,
            severityText: severityText,
            body: body,
            attributes: attributes,
            droppedAttributesCount: droppedAttributesCount,
            flags: UInt32(traceContext.traceFlags),
            traceID: traceContext.traceID,
            spanID: traceContext.spanID,
            eventName: eventName
        )
    }
}
```

- [ ] **Step 2: Run the failing test to verify it now passes**

```bash
swift test --filter "TraceContextIntegrationTests" 2>&1 | tail -10
```

Expected: 1 test passes. If you see "function 'init(...)' is ambiguous" or similar — the parameter signature is colliding with another initializer. Check the test argument labels match exactly: `timeUnixNano:` (UInt64), `severityNumber:`, `body:`, `traceContext:` — these must disambiguate from both the canonical `init` (which has `flags:` / `traceID:` / `spanID:` instead of `traceContext:`) and the `init(time: Time.Instant, ...)` (which has `time:` not `timeUnixNano:`).

- [ ] **Step 3: Run the full suite to confirm no regressions**

```bash
swift test 2>&1 | tail -3
```

Expected: All existing tests pass plus the 1 new one.

- [ ] **Step 4: Commit**

```bash
git add Sources/LogOTLP/TraceContextIntegration.swift
git commit -m "feat: OTLP.LogRecord(traceContext:) convenience init

Fills traceID/spanID/flags from an OTLP.TraceContext value.
Per OTLP spec, the low 8 bits of LogRecord.flags carry the W3C
traceFlags byte; high 24 bits remain zero."
```

---

## Task 4: Add the remaining test cases

**Files:**
- Modify: `/Users/satishbabariya/Desktop/swift-libs/swift-log-otlp/Tests/LogOTLPTests/TraceContextIntegrationTests.swift`

- [ ] **Step 1: Add 5 more tests covering edge cases**

Inside the `TraceContextIntegrationTests` struct (right after the `fillsCorrelationFields` test, before the closing `}`), append these tests:

```swift
    @Test("traceFlags=0 produces flags=0")
    func zeroFlagsPropagate() {
        let ctx = OTLP.TraceContext(
            traceID: Self.canonicalTraceID,
            spanID: Self.canonicalSpanID,
            traceFlags: 0x00
        )
        let record = OTLP.LogRecord(traceContext: ctx)
        #expect(record.flags == 0)
    }

    @Test("all 8 bits of traceFlags propagate into low 8 bits of flags")
    func allEightBitsPropagate() {
        let ctx = OTLP.TraceContext(
            traceID: Self.canonicalTraceID,
            spanID: Self.canonicalSpanID,
            traceFlags: 0xff
        )
        let record = OTLP.LogRecord(traceContext: ctx)
        #expect(record.flags == 0xff)
    }

    @Test("high 24 bits of flags stay zero")
    func highBitsZero() {
        let ctx = OTLP.TraceContext(
            traceID: Self.canonicalTraceID,
            spanID: Self.canonicalSpanID,
            traceFlags: 0x01
        )
        let record = OTLP.LogRecord(traceContext: ctx)
        #expect(record.flags & 0xffff_ff00 == 0)
    }

    @Test("defaults match the canonical init (everything zero/empty)")
    func defaultsMatchCanonical() {
        let ctx = OTLP.TraceContext(
            traceID: Self.canonicalTraceID,
            spanID: Self.canonicalSpanID,
            traceFlags: 0x01
        )
        let record = OTLP.LogRecord(traceContext: ctx)
        #expect(record.timeUnixNano == 0)
        #expect(record.observedTimeUnixNano == 0)
        #expect(record.severityNumber == .unspecified)
        #expect(record.severityText == "")
        #expect(record.body == nil)
        #expect(record.attributes.isEmpty)
        #expect(record.droppedAttributesCount == 0)
        #expect(record.eventName == "")
    }

    @Test("encodes to the same wire bytes as a hand-built canonical record")
    func wireRoundTrip() {
        let ctx = OTLP.TraceContext(
            traceID: Self.canonicalTraceID,
            spanID: Self.canonicalSpanID,
            traceFlags: 0x01
        )
        let viaContext = OTLP.LogRecord(
            timeUnixNano: 1_700_000_000_000_000_000,
            severityNumber: .info,
            body: .string("hello"),
            traceContext: ctx
        )
        let viaCanonical = OTLP.LogRecord(
            timeUnixNano: 1_700_000_000_000_000_000,
            severityNumber: .info,
            body: .string("hello"),
            flags: 0x01,
            traceID: Self.canonicalTraceID,
            spanID: Self.canonicalSpanID
        )
        let req1 = OTLP.ExportLogsServiceRequest(resourceLogs: [
            OTLP.ResourceLogs(
                scopeLogs: [OTLP.ScopeLogs(logRecords: [viaContext])]
            )
        ])
        let req2 = OTLP.ExportLogsServiceRequest(resourceLogs: [
            OTLP.ResourceLogs(
                scopeLogs: [OTLP.ScopeLogs(logRecords: [viaCanonical])]
            )
        ])
        #expect(OTLP.encodeLogs(req1) == OTLP.encodeLogs(req2))
    }
```

- [ ] **Step 2: Run the new tests**

```bash
swift test --filter "TraceContextIntegrationTests" 2>&1 | tail -12
```

Expected: 6 tests, all pass.

- [ ] **Step 3: Run the full suite**

```bash
swift test 2>&1 | tail -3
```

Expected: previous test count + 6, all pass.

- [ ] **Step 4: Commit**

```bash
git add Tests/LogOTLPTests/TraceContextIntegrationTests.swift
git commit -m "test: traceFlags propagation, defaults, wire-equivalence"
```

---

## Task 5: Update CHANGELOG

**Files:**
- Modify: `/Users/satishbabariya/Desktop/swift-libs/swift-log-otlp/CHANGELOG.md`

- [ ] **Step 1: Insert the 0.3.0 section**

In `CHANGELOG.md`, replace:

```markdown
## [Unreleased]

## [0.2.0] - 2026-05-10
```

with:

```markdown
## [Unreleased]

## [0.3.0] - 2026-05-13

### Added
- `OTLP.LogRecord.init(..., traceContext: OTLP.TraceContext, ...)` — convenience initializer that fills `traceID`, `spanID`, and `flags` from an `OTLP.TraceContext` value. Per OTLP spec the low 8 bits of `flags` carry the W3C `traceFlags` byte; high 24 bits remain zero.
- 6 new tests covering correlation-field propagation, all-bits propagation, high-bit zeroing, defaults, and wire-byte equivalence with the canonical initializer.

### Dependencies
- New: `swift-tracing-otlp` 0.3.0 — for the `OTLP.TraceContext` type.

### Migration
- Additive only. All v0.2 types, initializers, and helpers unchanged. The canonical `init(...)` with explicit `flags` / `traceID` / `spanID` fields continues to work.

### Phase 14
- Tranche 14C of [RFC-0019](https://github.com/bare-swift/bare-swift/blob/main/rfcs/0019-phase-14-anchor-otlp-cross-signal.md). Closes Phase 14 — OTLP cross-signal correlation now reachable from a single propagation value at the call site.

## [0.2.0] - 2026-05-10
```

- [ ] **Step 2: Commit**

```bash
git add CHANGELOG.md
git commit -m "docs: CHANGELOG 0.3.0"
```

---

## Task 6: Update README

**Files:**
- Modify: `/Users/satishbabariya/Desktop/swift-libs/swift-log-otlp/README.md`

- [ ] **Step 1: Read the current README**

```bash
sed -n '1,30p' README.md
```

Note the install version line (likely `from: "0.1.0"` or `from: "0.2.0"`).

- [ ] **Step 2: Bump install snippet version to 0.3.0**

Find the line:

```swift
.package(url: "https://github.com/bare-swift/swift-log-otlp.git", from: "0.2.0")
```

(it may say `0.1.0` — bump whatever is there to `0.3.0`)

Replace with:

```swift
.package(url: "https://github.com/bare-swift/swift-log-otlp.git", from: "0.3.0")
```

- [ ] **Step 3: Add a "Cross-signal correlation" section**

Before the existing `## Scope` section (or before the `## License` section if there's no `## Scope`), insert:

```markdown
## Cross-signal correlation

Since v0.3, `OTLP.LogRecord` accepts an `OTLP.TraceContext` (from [swift-tracing-otlp](https://github.com/bare-swift/swift-tracing-otlp)) to fill the W3C trace correlation fields in one shot:

```swift
import LogOTLP
import TracingOTLP
import OTLPExporter

let ctx = OTLP.TraceContext.parse(
    traceparent: "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
)!

let record = OTLP.LogRecord(
    timeUnixNano: 1_700_000_000_000_000_000,
    severityNumber: .info,
    body: .string("user logged in"),
    traceContext: ctx
)
// record.traceID, .spanID populated; record.flags == 0x01
```
```

(The triple-backtick fences in the inserted block close the surrounding fence pattern — make sure the `swift` block ends with its own closing ```` ``` ```` line and the surrounding markdown remains valid.)

- [ ] **Step 4: Update the Scope section if present**

If the README has a `## Scope` section, find the line listing what versions cover what (e.g., `**v0.2 covers:** ... Time.Instant convenience (v0.2).`) and append `W3C TraceContext convenience (v0.3).` so the scope reflects 0.3.

- [ ] **Step 5: Commit**

```bash
git add README.md
git commit -m "docs: README v0.3 install + cross-signal correlation example"
```

---

## Task 7: Update DocC catalog

**Files:**
- Modify: `/Users/satishbabariya/Desktop/swift-libs/swift-log-otlp/Sources/LogOTLP/Documentation.docc/LogOTLP.md`

- [ ] **Step 1: Read the current DocC file**

```bash
cat Sources/LogOTLP/Documentation.docc/LogOTLP.md
```

- [ ] **Step 2: Add a brief blurb (no cross-package symbol link)**

In the `## Topics` section, before the `### Errors` entry (or wherever it makes sense given the existing structure), insert this block:

```markdown
### Cross-signal correlation

`OTLP.LogRecord` accepts an `OTLP.TraceContext` (from swift-tracing-otlp) to fill `traceID` / `spanID` / `flags` in one call. See the README for usage.
```

**Note:** Do NOT use `` ``OTLPExporter/OTLP/TraceContext`` `` or any other DocC double-backtick reference to symbols from another bare-swift package — DocC strict mode will fail to resolve them. Use single-backtick `OTLP.TraceContext` as prose only. (See memory `feedback_docc_cross_package`.)

- [ ] **Step 3: Build DocC locally with warnings-as-errors**

```bash
swift package plugin --allow-writing-to-directory ./docs generate-documentation --target LogOTLP --output-path ./docs --transform-for-static-hosting --hosting-base-path swift-log-otlp --warnings-as-errors 2>&1 | tail -10
```

Expected: `Finished building documentation for 'LogOTLP'`. No warnings.

- [ ] **Step 4: Clean up the docs output (it's not committed)**

```bash
rm -rf docs
```

- [ ] **Step 5: Commit**

```bash
git add Sources/LogOTLP/Documentation.docc/LogOTLP.md
git commit -m "docs: DocC blurb for cross-signal correlation"
```

---

## Task 8: Ship v0.3.0

**Files:**
- (no file changes; git operations only)

- [ ] **Step 1: Push to main**

```bash
git push origin main
```

- [ ] **Step 2: Watch CI**

```bash
sleep 10 && gh run list --limit 1 --repo bare-swift/swift-log-otlp
```

Find the most recent in-progress run, then:

```bash
gh run watch <RUN_ID> --repo bare-swift/swift-log-otlp --exit-status 2>&1 | tail -10
```

Expected: All jobs ✓ (sanitizers, docc, build-and-test matrix on ubuntu-22.04 / ubuntu-24.04 / macos-15 across Swift 6.0 / 6.1). `api-stability` may be tag-conditional and skipped on main push — that's fine if every other job is ✓.

- [ ] **Step 3: Tag v0.3.0**

```bash
git tag -a v0.3.0 -m "swift-log-otlp 0.3.0 — OTLP.LogRecord(traceContext:) convenience init (Phase 14 Tranche 14C)"
git push origin v0.3.0
```

- [ ] **Step 4: Watch tag-triggered workflows**

```bash
sleep 10 && gh run list --limit 5 --repo bare-swift/swift-log-otlp
```

Find the `Release` workflow run for `v0.3.0`, then:

```bash
gh run watch <RELEASE_RUN_ID> --repo bare-swift/swift-log-otlp --exit-status 2>&1 | tail -8
```

Expected: `Release` and `Publish docs` workflows both green.

- [ ] **Step 5: Confirm release is published**

```bash
gh release view v0.3.0 --repo bare-swift/swift-log-otlp 2>&1 | head -10
```

Expected: `tag: v0.3.0`, `draft: false`, `prerelease: false`, URL present.

---

## Task 9: Bump umbrella index

**Files:**
- Modify: `/Users/satishbabariya/Desktop/swift-libs/bare-swift/packages/index.json`

- [ ] **Step 1: Locate the swift-log-otlp entry**

```bash
grep -n "swift-log-otlp" /Users/satishbabariya/Desktop/swift-libs/bare-swift/packages/index.json
```

- [ ] **Step 2: Bump the version**

In `/Users/satishbabariya/Desktop/swift-libs/bare-swift/packages/index.json`, find the block:

```json
    {
      "name": "swift-log-otlp",
      ...
      "version": "0.2.0",
      ...
    },
```

and change `"version": "0.2.0"` to `"version": "0.3.0"` (keep every other field unchanged).

- [ ] **Step 3: Commit and push from the umbrella repo**

```bash
cd /Users/satishbabariya/Desktop/swift-libs/bare-swift && git add packages/index.json && git commit -m "chore(index): swift-log-otlp 0.2.0 → 0.3.0 (Phase 14 14C)

OTLP.LogRecord(traceContext:) convenience init. RFC-0019 Phase 14 closes."
git push origin main
```

---

## Task 10: Close out 14C in memory

**Files:**
- Create: `/Users/satishbabariya/.claude/projects/-Users-satishbabariya-Desktop-swift-libs/memory/project_phase14_14c_shipped.md`
- Modify: `/Users/satishbabariya/.claude/projects/-Users-satishbabariya-Desktop-swift-libs/memory/MEMORY.md`

- [ ] **Step 1: Write the 14C memory file**

Create the file with exact content:

```markdown
---
name: phase-14-tranche-14c-complete
description: "swift-log-otlp v0.3.0 — OTLP.LogRecord(traceContext:) convenience init. 46 packages. Phase 14 CLOSED at 3/3."
metadata:
  type: project
---

**2026-05-13: swift-log-otlp v0.3.0 SHIPPED.** Closes Phase 14 per RFC-0019.

**How to apply:** Phase 14 fully closed. Next is "do gate 14" (Gate 14 retrospective → RFC-0020 anchoring Phase 15 — strongest candidate is swift-idna v0.2 UTS #46 mapping table). After gate, "ship 15a".

**Ecosystem: 46 packages (no count change; existing package version bump).**

**14C execution notes:**
- Single new source file: TraceContextIntegration.swift (~25 LOC, extension on OTLP.LogRecord).
- New convenience init takes `traceContext: OTLP.TraceContext` and fills `traceID`/`spanID`/`flags` (low 8 bits = traceFlags byte per OTLP spec; high 24 bits zero).
- 6 new tests: fillsCorrelationFields, zeroFlagsPropagate, allEightBitsPropagate, highBitsZero, defaultsMatchCanonical, wireRoundTrip.
- New dep: swift-tracing-otlp 0.3.0.
- DocC: avoided cross-package symbol link per `feedback_docc_cross_package` memory — used prose paragraph.

**RFC-0019 doc-error running list for Gate 14 retro:**
- 14A: package names wrong (swift-otlp-traces → swift-tracing-otlp, etc.).
- 14B: version wrong (called v0.2 but already at v0.2; needed v0.3).
- 14C: (fill in if any).

**Phase 14 progression — closed:**
- 14A shipped (swift-otlp-json v0.1.0).
- 14B shipped (swift-tracing-otlp v0.3.0).
- 14C shipped (swift-log-otlp v0.3.0 — this).

[[phase-14-tranche-14a-complete]] [[phase-14-tranche-14b-complete]]
```

- [ ] **Step 2: Update MEMORY.md index**

In `/Users/satishbabariya/.claude/projects/-Users-satishbabariya-Desktop-swift-libs/memory/MEMORY.md`, find the line:

```markdown
- [Phase 14 Tranche 14B complete](project_phase14_14b_shipped.md) — swift-tracing-otlp v0.3.0 (OTLP.TraceContext + W3C traceparent). Phase 14 at 2/3. 46 packages.
```

and insert this line **above** it:

```markdown
- [Phase 14 Tranche 14C complete](project_phase14_14c_shipped.md) — swift-log-otlp v0.3.0 (OTLP.LogRecord(traceContext:) init). Phase 14 CLOSED at 3/3. 46 packages.
```

- [ ] **Step 3: Done — report to user**

Print a short summary: package version, new init, test count delta, release URL, "Phase 14 closed; next is Gate 14 retro".

---

## Self-Review

- **Spec coverage:** Plan adds one convenience init (RFC-0019 14C deliverable), the dep that makes it possible (swift-tracing-otlp 0.3.0), tests covering the documented spec behavior (low 8 bits = traceFlags, high 24 = 0), CHANGELOG/README/DocC updates, ship + tag + umbrella bump + memory closure. All requirements covered.
- **Placeholder scan:** No TBDs, no "implement later", no "add appropriate validation" — every code step shows the exact code; every command step shows the exact command.
- **Type consistency:** `traceContext:` argument label used identically in Task 2 (failing test), Task 3 (implementation), Task 4 (further tests). `OTLP.TraceContext` (not `TraceContext`, not `TracingOTLP.TraceContext`) used everywhere. `UInt32(traceContext.traceFlags)` matches `traceFlags: UInt8` defined in 14B.

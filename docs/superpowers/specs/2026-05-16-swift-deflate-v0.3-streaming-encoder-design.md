# swift-deflate v0.3 streaming encoder — Design

**Phase 23 Tranche 23A** (RFC-0028). Existing package; additive non-breaking minor version bump.

**Date:** 2026-05-16

---

## Goal

Add a streaming DEFLATE encoder API (`Deflate.Streaming.Encoder` with `init(level:)` / `update(_:)` / `finish()`) on top of v0.2's one-shot `Deflate.encode(_:level:)`. Each `update(_:)` emits one DEFLATE block per chunk (`BFINAL=0`, BTYPE=10 dynamic-Huffman; or BTYPE=00 stored for `.none` level); `finish()` emits a terminator empty stored block (`BFINAL=1, BTYPE=00, LEN=0, NLEN=0xFFFF`). Closes documented v0.3 deferral from Phase 9. Continues codec-tier streaming sweep started in Phase 22; unblocks swift-gzip / swift-zlib / swift-content-encoding streaming in subsequent phases.

## Reality-check findings (locks the simpler design)

A pre-design survey of v0.2's encoder source (per Gate 22's canonical reality-check-before-RFC-estimate pattern):

1. **Existing `Deflate.Encoder` is "single-shot with streaming-shaped API."** Has `init(level:)` / `write(_:)` / `finish()`. Buffers all `write(_:)` input into a `Bytes` field; on `finish()` calls `Deflater(level:).encode(buffer)` — single one-shot encode. Comment explicitly says "Single-shot in v0.2. Streaming encoder ships as part of v0.3."
2. **`Deflater.encode()` emits one DEFLATE block per call** (or multiple stored blocks for `.none` since each stored block is capped at 65 535 bytes). `BlockEncoder.emitDynamic(tokens:, isFinal: Bool, writer:)` **already takes `isFinal: Bool`** — wire-compatible with non-final blocks.
3. **Matcher state is per-`Deflater.encode()` call.** Sliding 32 KiB window + LZ77 hash chains. **No state carried across encode calls in v0.2.** Per-chunk LZ77 only for streaming; window carry would be a v0.4 ratio improvement (matches swift-brotli v0.3 → v0.4 deferral pattern).
4. **`.default` / `.best` levels use 3-candidate pick-smallest** (stored / fixed Huffman / dynamic Huffman). **Cannot apply to streaming** — pick-smallest requires knowing the full block before emitting; streaming must commit per chunk. Streaming always emits dynamic-Huffman per chunk (or stored for `.none`). Slight ratio regression vs v0.2 one-shot accepted.
5. **DEFLATE has no stream header** (unlike Brotli's 4-bit WBITS header). The first block header is the stream start. Simplifies `init` — no header to emit.

**Conclusions:**
- Shape A applies (single tranche, per-chunk independent blocks, no window carry).
- Estimate: ~1-2 hours (same bracket as Phase 22 brotli streaming once state-carry was eliminated).
- **Existing `Deflate.Encoder` MUST be preserved unchanged** — `Deflate.encode(_:level:)` routes through it, and v0.2 byte-for-byte preservation is a non-negotiable contract.

## Scope (single tranche, streaming encoder + 1 error case)

**In scope:**
- `Deflate.Streaming` namespace enum.
- `Deflate.Streaming.Encoder` struct (Sendable, value-type).
- `init(level: Deflate.Encoder.Level = .default)` — non-throwing (Level is a closed enum; no runtime validation needed).
- `update(_ chunk: Bytes)` — emits one DEFLATE block per chunk (or N stored blocks for oversized `.none` chunks).
- `finish() throws(DeflateError) -> Bytes` — emits terminator empty stored block, returns accumulated bytes.
- 1 new `DeflateError` case: `encoderFinished` (mirror brotli's).
- ~15 new tests (round-trip + level coverage + boundary + error cases).

**Out of scope (deferred to v0.4+):**
- Sliding-window carry across chunks (Matcher state across `update(_:)` calls).
- Streaming inflate.
- `reset()` for encoder reuse.
- Explicit flush API.
- Multi-thread streaming.
- Block-type optimization (3-candidate pick-smallest in streaming).
- Fixed-Huffman streaming (`.fast` uses dynamic-Huffman in streaming for simplicity).

**Anti-goals:**
- No changes to v0.1 `Deflate.inflate(_:)` API surface.
- No changes to v0.2 `Deflate.encode(_:level:)` API surface or byte-for-byte output. Existing v0.2 round-trip tests verify regression.
- No changes to existing `Deflate.Encoder` struct. Preserved verbatim.
- No new dependencies (swift-bytes already in v0.1).
- No new core algorithms — only orchestration layer over existing primitives (Matcher, BitWriter, BlockEncoder.emitDynamic).

---

## File changes

```
swift-deflate/
  Package.swift                                ← unchanged
  Sources/Deflate/
    Deflate.swift                              ← add public enum Streaming namespace (~5 LOC)
    DeflateError.swift                         ← add encoderFinished case (~3 LOC)
    Streaming.swift                            ← NEW: Encoder struct + State enum (~150 LOC)
    Documentation.docc/Deflate.md              ← Streaming topic group + v0.3 overview (~20 LOC)
  Tests/DeflateTests/
    StreamingTests.swift                       ← NEW: ~15 tests (~250 LOC)
  CHANGELOG.md                                 ← v0.3.0 entry
  README.md                                    ← v0.3 install version + streaming example
```

**Critical regression guarantee:** existing `Deflate.Encoder` struct and `Deflate.encode(_:level:)` function are **preserved verbatim**. v0.2 byte-for-byte preservation is by construction (no code touched in the v0.2 path).

---

## Public API additions

```swift
extension Deflate {
    /// Streaming encoder namespace (v0.3+). For one-shot compression of
    /// bounded inputs, use ``Deflate/encode(_:level:)``.
    public enum Streaming: Sendable {}
}

extension Deflate.Streaming {
    /// Streaming DEFLATE encoder. Feed chunks via ``update(_:)`` and
    /// terminate with ``finish()``. The encoder emits one DEFLATE block
    /// per ``update(_:)`` call (dynamic-Huffman for non-`.none` levels;
    /// stored blocks for `.none`). Each block is independent — no LZ77
    /// match search crosses chunk boundaries in v0.3 (deferred to v0.4
    /// for ratio improvement).
    ///
    /// Usage:
    /// ```swift
    /// var encoder = Deflate.Streaming.Encoder(level: .default)
    /// encoder.update(chunk1)
    /// encoder.update(chunk2)
    /// let compressed = try encoder.finish()
    /// let plain = try Deflate.inflate(compressed)
    /// // plain == chunk1 + chunk2
    /// ```
    ///
    /// `Encoder` is a value type. Copying mid-stream produces two
    /// divergent encoders. Treat as single-owner.
    ///
    /// After ``finish()`` the encoder is in the finished state.
    /// ``update(_:)`` after finish is a silent no-op; double-finish throws
    /// ``DeflateError/encoderFinished``.
    public struct Encoder: Sendable {
        public init(level: Deflate.Encoder.Level = .default)
        public mutating func update(_ chunk: Bytes)
        public mutating func finish() throws(DeflateError) -> Bytes
    }
}

public enum DeflateError: Error, Equatable, Sendable {
    // ... existing 8 cases ...

    /// Encoder: ``Deflate/Streaming/Encoder/finish()`` was called twice on
    /// the same encoder.
    case encoderFinished
}
```

---

## Internals

### `Sources/Deflate/Streaming.swift` (NEW)

```swift
// SPDX-License-Identifier: Apache-2.0 WITH LLVM-exception
// Copyright (c) 2026 The bare-swift Project Authors.

import Bytes

extension Deflate.Streaming {
    public struct Encoder: Sendable {
        private enum State: Sendable {
            case open
            case finished
        }

        public let level: Deflate.Encoder.Level

        private var writer: BitWriter
        private var state: State

        public init(level: Deflate.Encoder.Level = .default) {
            self.level = level
            self.writer = BitWriter()
            self.state = .open
        }

        public mutating func update(_ chunk: Bytes) {
            guard case .open = state else { return }
            if chunk.isEmpty { return }

            switch level {
            case .none:
                emitStoredBlocks(chunk: chunk, isFinal: false)
            case .fast, .default, .best:
                emitDynamicBlock(chunk: chunk, isFinal: false)
            }
        }

        public mutating func finish() throws(DeflateError) -> Bytes {
            guard case .open = state else { throw .encoderFinished }
            state = .finished

            // Emit a terminator: empty stored block with BFINAL=1.
            // RFC 1951 § 3.2.4 — `1 00` + byte-align + LEN=0 + NLEN=0xFFFF.
            writer.writeBits(1, count: 1)  // BFINAL = 1
            writer.writeBits(0, count: 2)  // BTYPE  = 00 (stored)
            writer.alignToByte()
            writer.writeByte(0x00); writer.writeByte(0x00)  // LEN = 0
            writer.writeByte(0xFF); writer.writeByte(0xFF)  // NLEN = 0xFFFF

            return writer.finish()
        }

        // MARK: - Internal block emit

        /// Emit one dynamic-Huffman block for `chunk`. Runs LZ77 over the
        /// chunk only (no window carry).
        private mutating func emitDynamicBlock(chunk: Bytes, isFinal: Bool) {
            let tokens = collectTokens(chunk: chunk)
            BlockEncoder.emitDynamic(tokens: tokens, isFinal: isFinal, writer: &writer)
        }

        /// Emit one or more stored blocks covering `chunk`. RFC 1951 § 3.2.4
        /// stored blocks carry a 16-bit length, so oversized chunks split.
        private mutating func emitStoredBlocks(chunk: Bytes, isFinal: Bool) {
            let total = chunk.storage.count
            var pos = 0
            while pos < total {
                let blockSize = Swift.min(Deflater.storedBlockMax, total - pos)
                let isLastOfChunk = (pos + blockSize) == total
                let blockIsFinal = isFinal && isLastOfChunk

                writer.writeBits(blockIsFinal ? 1 : 0, count: 1)  // BFINAL
                writer.writeBits(0, count: 2)                      // BTYPE = 00 stored
                writer.alignToByte()
                let len = UInt16(blockSize)
                let nlen = ~len
                writer.writeByte(UInt8(truncatingIfNeeded: len & 0xFF))
                writer.writeByte(UInt8(truncatingIfNeeded: (len >> 8) & 0xFF))
                writer.writeByte(UInt8(truncatingIfNeeded: nlen & 0xFF))
                writer.writeByte(UInt8(truncatingIfNeeded: (nlen >> 8) & 0xFF))
                for i in 0..<blockSize {
                    writer.writeByte(chunk.storage[pos + i])
                }
                pos += blockSize
            }
        }

        /// Run LZ77 over `chunk` using `Matcher`. Per-chunk: matcher's hash
        /// state is reset per call (no window carry across chunks in v0.3).
        private func collectTokens(chunk: Bytes) -> [Token] {
            let maxChain: Int = {
                switch level {
                case .none: return 0
                case .fast: return 8
                case .default: return 32
                case .best: return 4096
                }
            }()
            var matcher = Matcher(chunk.storage, maxChain: maxChain)
            var tokens: [Token] = []
            let total = chunk.storage.count
            var pos = 0
            while pos < total {
                let (matchLen, matchDist) = matcher.findMatch(at: pos)
                if matchLen >= Matcher.minMatch {
                    tokens.append(.match(length: matchLen, distance: matchDist))
                    for k in 1..<matchLen where pos + k + Matcher.minMatch <= total {
                        _ = matcher.findMatch(at: pos + k)
                    }
                    pos += matchLen
                } else {
                    tokens.append(.literal(chunk.storage[pos]))
                    pos += 1
                }
            }
            return tokens
        }
    }
}
```

### `Sources/Deflate/Deflate.swift` (modify, ~5 LOC)

Append after the existing `extension Deflate { public static func encode(...) ... public struct Encoder ... }` block:

```swift
extension Deflate {
    /// Streaming encoder namespace (v0.3+). For one-shot compression of
    /// bounded inputs, use ``Deflate/encode(_:level:)``.
    public enum Streaming: Sendable {}
}
```

### `Sources/Deflate/DeflateError.swift` (modify, ~3 LOC)

Append before the closing `}` of the enum:

```swift
    /// Encoder: ``Deflate/Streaming/Encoder/finish()`` was called twice on
    /// the same encoder.
    case encoderFinished
```

---

## Stream-format details

### Single-chunk stream

```
update(chunk):
  [BFINAL=0, BTYPE=10, dynamic-Huffman block body covering chunk's tokens + EOB]

finish():
  [BFINAL=1, BTYPE=00, byte-align, LEN=0x0000, NLEN=0xFFFF]
```

Total: dynamic block + 5-byte stored terminator. Slightly larger than v0.2's `Deflate.encode(input)` one-shot because:
- v0.2 picks smallest of (stored / fixed Huffman / dynamic Huffman) for `.default`/`.best`; streaming always emits dynamic.
- v0.2 emits one final dynamic block; v0.3 streaming emits non-final dynamic block + 5-byte stored terminator.

**Not byte-equal to v0.2 one-shot.** Decoded equality holds.

### Empty stream

```
finish() only:
  [BFINAL=1, BTYPE=00, byte-align, LEN=0x0000, NLEN=0xFFFF]
```

Exactly 5 bytes: `0x01 0x00 0x00 0xFF 0xFF`.

**Byte-equal to `Deflate.encode(Bytes(), level: .none)`** (RFC 1951 § 3.2.4's empty-stored-block path that v0.2's `encodeStoredOnly` emits when `total == 0`). Regression-tested.

NOT byte-equal to `Deflate.encode(Bytes(), level: .default)` because v0.2's `.default` does 3-candidate pick-smallest and may pick fixed-Huffman empty block (different bytes).

### `.none` chunk stream

```
update(chunk) where level == .none:
  [BFINAL=0, BTYPE=00, byte-align, LEN=chunk.count, NLEN=~chunk.count, chunk bytes]
  (split into multiple stored blocks if chunk.count > 65535)

finish():
  [terminator]
```

---

## Test plan (~15 new tests)

### `Tests/DeflateTests/StreamingTests.swift` (NEW)

**Round-trip:**

1. `empty stream (no update + finish) round-trips to empty Bytes`.
2. `empty stream is byte-equal to Deflate.encode(empty, level: .none)` (5-byte terminator).
3. `single chunk update("hello") + finish round-trips to "hello"`.
4. `two chunks round-trip to concatenation`.
5. `100 × 1-byte chunks round-trip`.
6. `chunk > 64 KiB round-trips` (single dynamic block; not multi-block until chunk > some larger threshold based on token count, but a single dynamic block handles arbitrarily many tokens).
7. `mixed-size chunks (pangram + small + medium) round-trip`.
8. `empty chunk in middle is no-op`.

**Level coverage:**

9. `.none round-trip` (stored-only path).
10. `.fast round-trip`.
11. `.default round-trip`.
12. `.best round-trip`.

**Error cases:**

13. `double-finish throws encoderFinished`.

**Edge cases:**

14. `single-byte stream round-trips`.
15. `update after finish is silent no-op (then double-finish throws encoderFinished)`.

Total: 15 new tests in 1 new suite.

---

## CHANGELOG entry

```markdown
## [0.3.0] — 2026-05-16

### Added
- **Streaming encoder** — `Deflate.Streaming.Encoder` struct with `init(level:)` / `update(_:)` / `finish()`. Each `update(_:)` emits one DEFLATE block per chunk (dynamic-Huffman for non-`.none` levels; stored blocks for `.none`). `finish()` emits an empty-stored-block terminator (`BFINAL=1, BTYPE=00, LEN=0, NLEN=0xFFFF`).
- `Deflate.Streaming` public namespace enum.
- `DeflateError.encoderFinished` — thrown when `finish()` is called on an already-finished encoder.
- 15 new tests covering round-trip, all four levels, and error/edge cases.

### Dependencies
- No new dependencies. swift-bytes already in v0.1.

### Stream-format notes
- Streaming output is **valid DEFLATE** that decodes via the same `Deflate.inflate(_:)` v0.1 API.
- Empty stream output (`Deflate.Streaming.Encoder()` with no `update` calls + `finish()`) is **byte-equal** to `Deflate.encode(Bytes(), level: .none)`. Regression-tested.
- Single-update streaming output is **not** byte-equal to `Deflate.encode(_:)` one-shot output because (a) streaming always emits dynamic-Huffman per chunk (no 3-candidate pick-smallest), and (b) streaming adds a 5-byte stored-block terminator.
- No window carry across chunks in v0.3. LZ77 match search is per-chunk; matches across chunk boundaries are not found. Deferred to v0.4 for compression-ratio improvement.
- Streaming `.fast` / `.default` / `.best` all emit dynamic-Huffman blocks. The 3-candidate pick-smallest from v0.2 one-shot does not apply in streaming (requires future visibility).

### Migration (v0.2 → v0.3)
- **Additive only — non-breaking.** All v0.2 APIs unchanged.
- `Deflate.encode(_:level:)` continues to emit byte-equal output to v0.2 (regression-tested via existing v0.2 round-trip tests).
- `Deflate.Encoder` struct unchanged.
- `Deflate.inflate(_:)` unchanged from v0.1.
- `Deflate.Encoder.Level` unchanged.
- `DeflateError` adds 1 new case (additive; existing cases unchanged).

### Out of scope (deferred to v0.4+)
- Window carry across chunks (LZ77 across chunk boundaries — ratio improvement).
- Streaming inflate.
- Per-chunk explicit flush API.
- `reset()` for encoder reuse.
- Multi-threaded streaming.
- Block-type optimization (3-candidate pick-smallest in streaming).
- Fixed-Huffman streaming for `.fast` level.

### Phase 23
- Tranche 23A of [RFC-0028](https://github.com/bare-swift/bare-swift/blob/main/rfcs/0028-phase-23-anchor-swift-deflate-v0.3-streaming-encoder.md). Continues codec-tier streaming sweep (Phase 22 brotli → Phase 23 deflate → Phase 24+ gzip + zlib → Phase 25+ content-encoding wiring).
```

---

## README updates

- Bump install snippet `from: "0.2.0"` → `from: "0.3.0"`.
- Update tagline: "RFC 1951 DEFLATE codec — inflate (v0.1) + one-shot encode (v0.2) + streaming encode (v0.3). Sendable, Foundation-free."
- Add a `## Streaming (v0.3+)` section after the existing `## Usage`:

```swift
import Deflate
import Bytes

var encoder = Deflate.Streaming.Encoder(level: .default)
encoder.update(chunk1)
encoder.update(chunk2)
let compressed = try encoder.finish()
let plain = try Deflate.inflate(compressed)
// plain == chunk1 + chunk2
```

- Add a "Public API" line for `Deflate.Streaming.Encoder`.

---

## DocC catalog updates

In `Sources/Deflate/Documentation.docc/Deflate.md`:

- Update Overview to mention v0.3 streaming.
- Add a `### Streaming compress (v0.3+)` topic group:

```
### Streaming compress (v0.3+)

- ``Deflate/Streaming``
- ``Deflate/Streaming/Encoder``
```

---

## CI considerations

No CI workflow changes. swift-deflate's existing reusable workflow handles the build/test/sanitizers/docc matrix.

---

## Migration

**Additive only — non-breaking.** v0.2 callers' code compiles and behaves identically against v0.3:
- `Deflate.encode(input)` continues to produce byte-equal output.
- `Deflate.Encoder(level:).write(input).finish()` continues to produce byte-equal output (v0.2 path unchanged).
- `Deflate.inflate(bytes)` unchanged.

Adopters opt into streaming by switching to `Deflate.Streaming.Encoder`.

---

## Ship sequencing

1. Modify `DeflateError.swift` (add `encoderFinished` case).
2. Check tests for `switch.*DeflateError` exhaustiveness patterns; update if needed (per Gate 22 memory note).
3. Modify `Deflate.swift` (add `public enum Streaming: Sendable {}` namespace declaration).
4. Create `Sources/Deflate/Streaming.swift` (Encoder struct + State enum + per-chunk block emission).
5. Run v0.2 tests — verify byte-for-byte regression.
6. Create `Tests/DeflateTests/StreamingTests.swift` (~15 tests).
7. Run full test suite.
8. Update CHANGELOG + README + DocC.
9. Build DocC locally with `--warnings-as-errors`.
10. Commit, push, watch CI.
11. Tag v0.3.0; watch Release + Publish docs.
12. Umbrella bump (`swift-deflate` 0.2.0 → 0.3.0 in `packages/index.json`).
13. Memory closure (`project_phase23_23a_shipped.md`).

Total Phase 23 calendar: ~1-2 hours wall-clock per reality-check finding.

---

## Decisions log (locked)

- **NEW `Deflate.Streaming.Encoder` parallel namespace.** Existing `Deflate.Encoder` preserved verbatim. Two coexisting encoder types — docstrings clarify when to use which.
- **Per-chunk DEFLATE block emission.** One dynamic-Huffman block per `update(_:)` call. `.none` level uses stored blocks (potentially multiple per chunk if chunk > 65 535 bytes).
- **`finish()` emits empty stored block terminator** (5 bytes: `0x01 0x00 0x00 0xFF 0xFF`). Simplest valid stream-terminator path; matches v0.2's empty-input behavior for `.none`.
- **No window carry across chunks in v0.3.** Each chunk's Matcher state is fresh. Deferred to v0.4.
- **Streaming always emits dynamic-Huffman per chunk for non-`.none` levels.** 3-candidate pick-smallest from v0.2 one-shot path doesn't apply. Slight ratio regression accepted.
- **`init` is non-throwing.** Level is a closed enum (no `.level(Int)` like brotli's Quality); no runtime validation needed.
- **`update(_:)` is non-throwing.** Empty chunk = no-op. Update-after-finish = silent no-op.
- **`finish()` is mutating + throwing.** Throws `encoderFinished` on double-call.
- **Empty stream → byte-equal to `Deflate.encode(Bytes(), level: .none)`.** Strong regression signal. Test guarantees this.
- **`Deflate.encode(_:level:)` byte-for-byte preservation** by construction (v0.2 code path untouched).
- **Sendable struct + State enum.** Mirrors Phase 22 brotli template.
- **No DocC cross-package symbol links.** Per global memory note.

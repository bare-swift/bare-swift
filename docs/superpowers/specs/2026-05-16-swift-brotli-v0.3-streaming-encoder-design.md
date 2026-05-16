# swift-brotli v0.3 streaming encoder — Design

**Phase 22 Tranche 22A** (RFC-0027). Existing package; additive non-breaking minor version bump.

**Date:** 2026-05-16

---

## Goal

Add a streaming Brotli encoder API (`Brotli.Streaming.Encoder` with `init(quality:)` / `update(_:)` / `finish()`) on top of v0.2's one-shot `Brotli.compress(_:quality:)`. Each `update(_:)` emits one non-last metablock per chunk; `finish()` emits a 2-bit terminator metablock and finalizes. Closes the longest-standing codec-tier deferral (9 deferrals since Phase 12). Non-breaking — all v0.1 decode + v0.2 compress APIs unchanged byte-for-byte.

## Reality-check findings (informs the simpler-than-RFC design)

A pre-design survey of v0.2's encoder source revealed that v0.3 streaming is **simpler than RFC-0027's "Shape A" sketched**:

1. **v0.2 carries no state across metablocks.** `EncoderMetaBlock.emit` builds fresh Huffman trees per metablock; uses direct distance codes (no last-4-distance ring-buffer shortcuts); `MatchFinder.scan` is stateless. RFC 7932 § 4's distance ring-buffer carry is **encoder-optional** — decoder maintains it for shortcuts, but a v0.2-style encoder that uses direct codes never benefits from it.
2. **No window carry needed for correctness.** RFC 7932 allows non-last metablocks (ISLAST=0) that reset trees independently. Streaming = sequence of independent metablocks. Window carry is a **ratio optimization**, not a correctness requirement.
3. **BitWriter is correctly stateful across multiple metablock emits.** Already accumulates bits across writes; only one `alignToByte()` at stream end. Streaming uses BitWriter as-is.
4. **`EncoderMetaBlock.emit` hardcodes ISLAST=1.** Needs `isLast: Bool` parameter; non-last path emits ISUNCOMPRESSED=0 (currently elided because "decoder reads it only when !isLast").

**Conclusion:** Window carry across chunks is deferred to v0.4 (ratio optimization). v0.3 is "sequence of independent metablocks" — correct, simple, slightly worse ratio across chunk boundaries.

## Scope (single tranche, streaming encoder + 1 error case)

**In scope:**
- `Brotli.Streaming` namespace enum.
- `Brotli.Streaming.Encoder` struct (Sendable, value-type).
- `init(quality: Brotli.Quality = .default) throws(BrotliError)` — emits stream header.
- `update(_ chunk: Bytes)` — emits one metablock per chunk (or N metablocks if chunk > 16 MiB).
- `finish() throws(BrotliError) -> Bytes` — emits 2-bit terminator, byte-aligns, returns accumulated bytes.
- `EncoderMetaBlock.emit` extended with `isLast: Bool` parameter.
- 1 new `BrotliError` case: `encoderFinished` (for double-finish).
- ~15 new tests (round-trip + boundary + error cases).

**Out of scope (deferred to v0.4+):**
- Window carry across chunks (LZ77 match search spans chunk boundaries).
- Per-chunk explicit flush API (e.g., `flushAndAlign()`).
- `reset()` for encoder reuse.
- Streaming decode (Phase 23+ candidate).
- Static-dictionary search (unchanged from v0.2).
- Multi-thread streaming (per-metablock parallelization).
- `unfinished()` queries (encoder state inspection).

**Anti-goals:**
- No changes to v0.1 decode API surface.
- No changes to v0.2 compress API surface or byte-for-byte output. Regression test verifies single `Brotli.compress(_:)` call produces identical output before and after.
- No new dependencies (swift-bytes already in v0.1).
- No new core algorithms — only orchestration layer over existing primitives.

---

## File changes

```
swift-brotli/
  Package.swift                                ← unchanged
  Sources/Brotli/
    Brotli.swift                               ← add public enum Streaming declaration (~5 LOC)
    Streaming.swift                            ← NEW: Encoder struct + State enum (~100 LOC)
    EncoderMetaBlock.swift                     ← add isLast: Bool param; non-last path emits ISUNCOMPRESSED=0 (~15 LOC change)
    Encoder.swift                              ← pass isLast: true to maintain v0.2 byte-for-byte (~1 LOC)
    BrotliError.swift                          ← add encoderFinished case (~3 LOC)
    Documentation.docc/Brotli.md               ← add Streaming topic group (~20 LOC)
  Tests/BrotliTests/
    StreamingTests.swift                       ← NEW: ~15 tests (~250 LOC)
  CHANGELOG.md                                 ← v0.3.0 entry
  README.md                                    ← install version + streaming example
```

Total new source: ~125 LOC. Tests: ~250 LOC. Docs: ~50 LOC.

---

## Public API additions

```swift
extension Brotli {
    /// Streaming encoder namespace (v0.3+). For one-shot compression of
    /// bounded inputs ≤16 MiB, use ``Brotli/compress(_:quality:)``.
    public enum Streaming: Sendable {
        /// Streaming Brotli encoder. Feed chunks via ``update(_:)`` and
        /// terminate with ``finish()``. The encoder emits one metablock per
        /// ``update(_:)`` call (or N metablocks if a chunk exceeds 16 MiB).
        /// Each metablock is independent — no LZ77 match search crosses
        /// chunk boundaries in v0.3 (deferred to v0.4 for ratio improvement).
        ///
        /// Usage:
        /// ```swift
        /// var encoder = try Brotli.Streaming.Encoder(quality: .default)
        /// encoder.update(chunk1)
        /// encoder.update(chunk2)
        /// let compressed = try encoder.finish()
        /// let plain = try Brotli.decode(compressed)
        /// // plain == chunk1 + chunk2
        /// ```
        ///
        /// `Encoder` is a value type. Copying mid-stream produces two
        /// divergent encoders — each accumulates its own bytes from the
        /// copy point. This is rarely useful; treat the encoder as
        /// single-owner.
        ///
        /// After ``finish()`` the encoder is in the finished state.
        /// Subsequent ``update(_:)`` calls are silent no-ops. Subsequent
        /// ``finish()`` calls throw ``BrotliError/encoderFinished``.
        public struct Encoder: Sendable {
            public init(quality: Brotli.Quality = .default) throws(BrotliError)

            /// Feed a chunk to the encoder. Emits one metablock per call
            /// (or N metablocks if `chunk.count > Brotli.maxInputSize`).
            /// Empty chunk = no-op (no metablock emitted).
            /// No-op when called after ``finish()``.
            public mutating func update(_ chunk: Bytes)

            /// Emit the terminator metablock, byte-align, return accumulated
            /// stream bytes. Throws ``BrotliError/encoderFinished`` on
            /// double-call.
            public mutating func finish() throws(BrotliError) -> Bytes
        }
    }
}

public enum BrotliError: Error, Equatable, Sendable {
    // ... existing 12 cases ...

    /// Encoder: ``Brotli/Streaming/Encoder/finish()`` was called twice on
    /// the same encoder.
    case encoderFinished
}
```

---

## Internals

### `Sources/Brotli/Streaming.swift` (NEW)

```swift
// SPDX-License-Identifier: Apache-2.0 WITH LLVM-exception
// Copyright (c) 2026 The bare-swift Project Authors.

import Bytes

extension Brotli.Streaming {
    public struct Encoder: Sendable {
        private enum State: Sendable {
            case open
            case finished
        }

        private let quality: Brotli.Quality
        private var writer: BitWriter
        private var state: State

        public init(quality: Brotli.Quality = .default) throws(BrotliError) {
            let q = quality.rawValue
            guard q >= 0 && q <= 11 else { throw .qualityOutOfRange }
            self.quality = quality
            self.writer = BitWriter()
            self.state = .open

            // Emit stream header (WBITS=22). Same 4 bits as Encoder.encode.
            self.writer.writeBit(1)
            self.writer.writeBits(5, count: 3)
        }

        public mutating func update(_ chunk: Bytes) {
            guard case .open = state else { return }
            if chunk.isEmpty { return }

            // Split chunks > 16 MiB into multiple metablocks.
            let cap = Brotli.maxInputSize
            var offset = 0
            let total = chunk.count
            while offset < total {
                let end = min(offset + cap, total)
                let subChunk = Bytes(Array(chunk)[offset..<end])
                let commands = MatchFinder.scan(subChunk, quality: quality)
                EncoderMetaBlock.emit(
                    commands: commands,
                    inputSize: subChunk.count,
                    isLast: false,
                    to: &writer
                )
                offset = end
            }
        }

        public mutating func finish() throws(BrotliError) -> Bytes {
            guard case .open = state else { throw .encoderFinished }
            state = .finished

            // Terminator metablock: ISLAST=1, ISLASTEMPTY=1 (2 bits).
            EncoderMetaBlock.emit(
                commands: [],
                inputSize: 0,
                isLast: true,
                to: &writer
            )

            writer.alignToByte()
            return writer.finalize()
        }
    }
}
```

### `Sources/Brotli/EncoderMetaBlock.swift` (modify)

Add `isLast: Bool` parameter. Refactor existing two paths:

```swift
enum EncoderMetaBlock {
    static func emit(commands: [EncoderCommand], inputSize: Int, isLast: Bool, to w: inout BitWriter) {
        // ISLAST
        w.writeBit(isLast ? 1 : 0)

        // ISLASTEMPTY follows only when ISLAST=1.
        if isLast {
            if inputSize == 0 {
                w.writeBit(1)  // ISLASTEMPTY = 1; metablock done.
                return
            }
            w.writeBit(0)  // ISLASTEMPTY = 0
        }

        // For non-last: MLEN follows directly. For last (non-empty): same.
        // ... existing MNIBBLES + MLEN emit ...

        // ISUNCOMPRESSED is read by decoder only when !isLast. For non-last
        // metablocks we ALWAYS use compressed mode, so emit 0.
        if !isLast {
            w.writeBit(0)  // ISUNCOMPRESSED = 0
        }

        // ... rest of existing trees + commands emit unchanged ...
    }
}
```

### `Sources/Brotli/Encoder.swift` (modify, 1 line)

```swift
// Existing call site:
EncoderMetaBlock.emit(commands: commands, inputSize: bytes.count, to: &w)
// Becomes:
EncoderMetaBlock.emit(commands: commands, inputSize: bytes.count, isLast: true, to: &w)
```

This preserves v0.2 byte-for-byte output (ISLAST=1 path is unchanged from existing logic).

### `Sources/Brotli/BrotliError.swift` (modify, ~3 LOC)

Add `case encoderFinished` after the existing v0.2 cases.

### `Sources/Brotli/Brotli.swift` (modify, ~5 LOC)

Add `public enum Streaming: Sendable {}` namespace declaration. Body of `Encoder` lives in `Streaming.swift` via `extension Brotli.Streaming`.

---

## Stream-format details

### Byte-by-byte structure of a v0.3 streaming output

Stream header (4 bits, LSB-first):
```
bit 0: 1 (first stream-header bit)
bits 1-3: 5 in LSB-first = 1, 0, 1
```

Per chunk (non-last metablock):
```
bit: ISLAST = 0
bits: MNIBBLES code (2 bits: 0/1/2 for 4/5/6 nibbles)
bits: MLEN-1 (4*MNIBBLES bits)
bit: ISUNCOMPRESSED = 0
bit: NBLTYPESL = 1 (single bit '0')
bit: NBLTYPESI = 1 (single bit '0')
bit: NBLTYPESD = 1 (single bit '0')
bits: NPOSTFIX = 0 (2 bits)
bits: NDIRECT = 0 (4 bits)
bits: CMODE = 0 (2 bits for the one literal block type)
bit: NTREESL = 1 (single bit '0')
bit: NTREESD = 1 (single bit '0')
... [3 prefix-code descriptors + command stream — same as v0.2] ...
```

Terminator (2 bits):
```
bit: ISLAST = 1
bit: ISLASTEMPTY = 1
```

Final byte-align: pad with zeros to byte boundary.

### Empty-stream output equivalence with v0.2

v0.2's `Brotli.compress(Bytes())` emits:
```
header (4 bits) + ISLAST=1 + ISLASTEMPTY=1 = 6 bits → byte-align → 1 byte
```

v0.3's `Brotli.Streaming.Encoder()` with no `update` calls + `finish()` emits:
```
header (4 bits) + ISLAST=1 + ISLASTEMPTY=1 = 6 bits → byte-align → 1 byte
```

**Byte-equal.** Regression test asserts this.

### Single-chunk vs v0.2 one-shot

For a non-empty single chunk, v0.3 emits:
- v0.2: `[header 4 bits] [ISLAST=1, ISLASTEMPTY=0, MLEN, ...trees..., commands]`
- v0.3: `[header 4 bits] [ISLAST=0, MLEN, ISUNCOMPRESSED=0, ...trees..., commands] [ISLAST=1, ISLASTEMPTY=1]`

Difference: v0.3 adds 1 ISUNCOMPRESSED bit + 2 terminator bits + lacks ISLASTEMPTY bit = net +2 bits per stream. After byte alignment, this is at most +1 byte. **Not byte-equal**, but valid Brotli.

---

## Test plan (~15 new tests)

### `Tests/BrotliTests/StreamingTests.swift` (NEW)

**Round-trip:**

1. `empty stream (no update + finish) round-trips through Brotli.decode to empty Bytes` — also asserts byte-equal to v0.2's `Brotli.compress(Bytes())`.
2. `single chunk update("hello") + finish round-trips to "hello"`.
3. `two chunks update("hel") + update("lo") + finish round-trips to "hello"`.
4. `100 × 1-byte chunks round-trip to original 100-byte input`.
5. `single chunk ≥16 MiB triggers internal multi-metablock split + round-trips`.
6. `mixed-size chunks (pangram + small + medium) round-trip`.
7. `empty chunk in middle is no-op: update("a") + update(empty) + update("b") + finish → "ab"`.

**Quality coverage:**

8. `.fastest quality round-trip (literal-only path)`.
9. `.smallest quality round-trip (deepest match search)`.

**Equivalence:**

10. `single-update output decodes to same bytes as Brotli.compress one-shot output (decoded equality)`.

**Error cases:**

11. `init(quality: .level(-1)) throws qualityOutOfRange`.
12. `init(quality: .level(12)) throws qualityOutOfRange`.
13. `double-finish throws encoderFinished`.

**Edge cases:**

14. `single-byte stream round-trips`.
15. `update after finish is silent no-op (then double-finish throws encoderFinished)`.

Total: 15 new tests. Total package tests post-22A: ~103 + 15 = ~118.

---

## CHANGELOG entry

```markdown
## [0.3.0] — 2026-05-16

### Added
- **Streaming encoder** — `Brotli.Streaming.Encoder` struct with `init(quality:)` / `update(_:)` / `finish()`. Each `update(_:)` emits one non-last metablock per chunk (or N metablocks if a chunk exceeds 16 MiB). `finish()` emits a 2-bit terminator metablock and returns accumulated bytes.
- `Brotli.Streaming` public namespace enum.
- `BrotliError.encoderFinished` — thrown when `finish()` is called on an already-finished encoder.
- ~15 new tests covering round-trip (empty, single chunk, two chunks, many tiny chunks, ≥16 MiB chunks), quality coverage (`.fastest`, `.smallest`), and error/edge cases (invalid quality at init, double-finish, update-after-finish no-op).

### Dependencies
- No new dependencies. swift-bytes already in v0.1.

### Stream-format notes
- Streaming output is **valid Brotli** that decodes via the same `Brotli.decode(_:)` v0.1 API.
- Empty stream output (`Brotli.Streaming.Encoder()` with no `update` calls + `finish()`) is **byte-equal** to `Brotli.compress(Bytes())`. Regression-tested.
- Single-update streaming output is **not** byte-equal to `Brotli.compress(_:)` one-shot output (differs by ~1-3 bits due to extra ISUNCOMPRESSED bit + terminator metablock). Decoded equality holds.
- No window carry across chunks in v0.3. LZ77 match search is per-chunk; matches across chunk boundaries are not found. Defer to v0.4 for compression-ratio improvement.

### Migration (v0.2 → v0.3)
- **Additive only — non-breaking.** All v0.2 APIs unchanged.
- `Brotli.compress(_:quality:)` continues to work; emits byte-equal output to v0.2 (regression-tested).
- `Brotli.decode(_:)` unchanged from v0.1.
- `Brotli.Quality` unchanged.
- `BrotliError` adds 1 new case (additive; existing cases unchanged).

### Out of scope (deferred to v0.4+)
- Window carry across chunks (LZ77 across chunk boundaries — ratio improvement).
- Per-chunk explicit flush API.
- `reset()` for encoder reuse.
- Streaming decode.
- Static-dictionary search.
- Multi-threaded streaming.

### Phase 22
- Tranche 22A of [RFC-0027](https://github.com/bare-swift/bare-swift/blob/main/rfcs/0027-phase-22-anchor-swift-brotli-v0.3-streaming-encoder.md). Closes the longest-standing codec-tier deferral (9 consecutive gates since Phase 12).
```

---

## README updates

- Bump install snippet `from: "0.1.0"` (or `0.2.0` — check current) to `from: "0.3.0"`.
- Update tagline: "RFC 7932 Brotli codec — decoder (v0.1) + encoder (v0.2 one-shot, v0.3 streaming)".
- Add a `## Streaming (v0.3+)` section between `## Usage` and `## Scope`:

```swift
import Brotli
import Bytes

var encoder = try Brotli.Streaming.Encoder(quality: .default)
encoder.update(chunk1)
encoder.update(chunk2)
let compressed = try encoder.finish()
let plain = try Brotli.decode(compressed)
// plain == chunk1 + chunk2
```

- Update Scope section to note v0.3 adds streaming encoder; defer v0.4 items (window carry, flush, reset) to Out-of-scope.

---

## DocC catalog updates

In `Sources/Brotli/Documentation.docc/Brotli.md`:

- Update Overview to mention v0.3 ships streaming encoder.
- Add `### Streaming (v0.3+)` topic group:

```
### Streaming (v0.3+)

- ``Brotli/Streaming``
- ``Brotli/Streaming/Encoder``
```

- Update top-level "supported operations" prose to list decode + one-shot compress + streaming compress.

---

## CI considerations

No CI workflow changes. Existing reusable workflow handles the build/test/sanitizers/docc matrix. Sanitizers stay ON (no large new static data — Streaming.swift is small).

---

## Migration

**Additive only — non-breaking.** v0.2 callers' code compiles and behaves identically against v0.3:
- `Brotli.compress(bytes)` continues to work for ≤16 MiB inputs.
- `Brotli.decode(bytes)` unchanged.
- All v0.2 tests pass against v0.3 (regression).

Adopters opt into streaming by switching to `Brotli.Streaming.Encoder`.

---

## Ship sequencing

1. Modify `EncoderMetaBlock.swift` (add `isLast: Bool` param; handle ISUNCOMPRESSED for non-last path).
2. Modify `Encoder.swift` (pass `isLast: true` to existing call site — v0.2 byte-for-byte preservation).
3. Modify `BrotliError.swift` (add `encoderFinished` case).
4. Modify `Brotli.swift` (add `public enum Streaming: Sendable {}` namespace declaration).
5. Create `Sources/Brotli/Streaming.swift` (Encoder struct + State enum).
6. Run v0.2 tests — verify byte-for-byte regression.
7. Create `Tests/BrotliTests/StreamingTests.swift` (~15 tests).
8. Run full test suite (~118 tests).
9. Update CHANGELOG + README + DocC.
10. Build DocC locally with `--warnings-as-errors`.
11. Commit, push, watch CI.
12. Tag v0.3.0; watch Release + Publish docs.
13. Umbrella bump (`swift-brotli` 0.2.0 → 0.3.0 in `packages/index.json`).
14. Memory closure (`project_phase22_22a_shipped.md`).

Total Phase 22 calendar: ~2-3 hours wall-clock (under RFC-0027's 3-5 hour bracket; the reality-check simplification eliminated window-carry complexity).

---

## Decisions log (locked)

- **Each `update(_:)` emits one metablock with `ISLAST=0`.** Per-chunk metablocks; no buffering between calls. Internal split into 16 MiB sub-metablocks for oversized chunks.
- **`finish()` emits a 2-bit terminator metablock** (ISLAST=1, ISLASTEMPTY=1). Simplest valid stream terminator. Adds ~2 bits + byte-alignment to stream size.
- **No window carry across chunks in v0.3.** Each metablock independent; per-chunk LZ77. Deferred to v0.4.
- **`update(_:)` is non-throwing.** Throws would force `try` at every per-chunk call site. After finish, update is silent no-op.
- **`finish()` is mutating + throwing.** Throws `encoderFinished` on double-call. Not `consuming` — keeps Encoder Copyable for value-type semantics (with documented copy-divergence behavior).
- **`init` is throwing.** Validates quality at init (fail-early). `qualityOutOfRange` for invalid `Quality.level(n)` values.
- **Empty chunk → no-op.** Empty `update(_:)` emits no metablock. Saves ~1 byte per empty call.
- **Encoder is Sendable, value-type, Copyable.** Documented: copying mid-stream produces two divergent encoders. Rarely useful; treat as single-owner.
- **No new error cases beyond `encoderFinished`.** Reuse existing `qualityOutOfRange`. `inputTooLarge` is no longer thrown by streaming (oversized chunks split internally).
- **Empty-stream output is byte-equal to v0.2.** Regression test confirms. Saves implementation surprise.
- **v0.2 one-shot byte-for-byte preservation.** Encoder.swift passes `isLast: true` to the modified `EncoderMetaBlock.emit`, which falls through to the existing (unchanged) ISLAST=1 path. Regression-tested via existing v0.2 test suite.
- **DocC `Brotli/Streaming/Encoder` path** uses nested-type DocC convention. No cross-package symbol links per global memory note.

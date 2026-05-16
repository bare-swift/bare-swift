# swift-gzip v0.3 streaming encoder — Design

**Phase 24 Tranche 24A** (RFC-0029). Existing package; additive non-breaking minor version bump.

**Date:** 2026-05-16

## Goal

Add `Gzip.Streaming.Encoder` (init/update/finish) on top of v0.2's one-shot `Gzip.encode(_:level:filename:modificationTime:)`. Each `update(_:)` feeds the chunk to an inner `Deflate.Streaming.Encoder` (Phase 23A) + updates an incremental `CRC.Digest<UInt32>` over the uncompressed bytes + accumulates the ISIZE byte counter. `finish()` finalizes the inner DEFLATE, appends the 8-byte CRC32 LE + ISIZE LE trailer. Mirrors Phase 23 deflate streaming template exactly.

## Reality-check findings

- **swift-crc has `CRC.Digest<Width>` incremental API** (`init(algorithm:)` + `mutating func update(_:)` + `func finalize() -> Width`). Use directly — no swift-crc changes needed.
- **swift-deflate v0.3** ships `Deflate.Streaming.Encoder` (Phase 23A, just landed).
- **gzip v0.2 `Encoder` struct** has `(level, filename, modificationTime)` params. **Preserved verbatim** per Gate 23 existing-encoder-preserved-verbatim pattern.
- **`GzipError.malformedDeflate(DeflateError)`** already exists for wrapping inner errors. Add `GzipError.encoderFinished` for double-finish.
- Header-build logic in v0.2's `GzipEncoder.encode` can be lifted into a small helper that streaming reuses.
- **Existing exhaustive switch** at `Tests/GzipTests/GzipTests.swift:365` requires update for new error case.

**Conclusion:** Same template as Phase 23. Estimate ~1 hour wall-clock.

## File changes

```
swift-gzip/
  Sources/Gzip/
    Gzip.swift                                ← +public enum Streaming namespace (~5 LOC)
    GzipError.swift                           ← +encoderFinished case (~3 LOC)
    Streaming.swift                           ← NEW: Encoder struct + State enum (~140 LOC)
    Documentation.docc/Gzip.md                ← Streaming topic group (~20 LOC)
  Tests/GzipTests/
    GzipTests.swift                           ← extend exhaustive switch (~1 LOC)
    StreamingTests.swift                      ← NEW: 15 tests (~240 LOC)
  CHANGELOG.md                                ← v0.3.0 entry
  README.md                                   ← v0.3 install + streaming example
  Package.swift                               ← swift-deflate dep bump 0.2.0 → 0.3.0
```

## Public API additions

```swift
extension Gzip {
    public enum Streaming: Sendable {}
}

extension Gzip.Streaming {
    public struct Encoder: Sendable {
        public typealias Level = Deflate.Encoder.Level
        public init(level: Level = .default, filename: String? = nil, modificationTime: UInt32 = 0)
        public mutating func update(_ chunk: Bytes)
        public mutating func finish() throws(GzipError) -> Bytes
    }
}

public enum GzipError: Error, Equatable, Sendable {
    // ... existing 10 cases ...
    case encoderFinished
}
```

## Internals

`Sources/Gzip/Streaming.swift`:

```swift
import Bytes
import Deflate
import CRC

extension Gzip.Streaming {
    public struct Encoder: Sendable {
        public typealias Level = Deflate.Encoder.Level

        private enum State: Sendable { case open; case finished }

        public let level: Level
        public let filename: String?
        public let modificationTime: UInt32

        private var headerBytes: ContiguousArray<UInt8>  // pre-built; emitted first on finish
        private var deflateEncoder: Deflate.Streaming.Encoder
        private var crc: CRC.Digest<UInt32>
        private var isize: UInt64
        private var state: State

        public init(level: Level = .default, filename: String? = nil, modificationTime: UInt32 = 0) {
            self.level = level
            self.filename = filename
            self.modificationTime = modificationTime
            self.headerBytes = Self.buildHeader(level: level, filename: filename, modificationTime: modificationTime)
            self.deflateEncoder = Deflate.Streaming.Encoder(level: level)
            self.crc = CRC.Digest(algorithm: .iso_hdlc)
            self.isize = 0
            self.state = .open
        }

        public mutating func update(_ chunk: Bytes) {
            guard case .open = state else { return }
            if chunk.isEmpty { return }
            deflateEncoder.update(chunk)
            crc.update(chunk.storage)
            isize &+= UInt64(chunk.storage.count)
        }

        public mutating func finish() throws(GzipError) -> Bytes {
            guard case .open = state else { throw .encoderFinished }
            state = .finished

            // Finalize inner DEFLATE. Wrap potential errors as malformedDeflate.
            let deflateBytes: Bytes
            do {
                deflateBytes = try deflateEncoder.finish()
            } catch {
                throw .malformedDeflate(error)
            }

            // Assemble: header + deflate + crc32 (LE) + isize (LE, low 32 bits).
            var out = ContiguousArray<UInt8>()
            out.reserveCapacity(headerBytes.count + deflateBytes.storage.count + 8)
            out.append(contentsOf: headerBytes)
            out.append(contentsOf: deflateBytes.storage)
            let crcValue = crc.finalize()
            out.append(UInt8(truncatingIfNeeded: crcValue & 0xFF))
            out.append(UInt8(truncatingIfNeeded: (crcValue >> 8) & 0xFF))
            out.append(UInt8(truncatingIfNeeded: (crcValue >> 16) & 0xFF))
            out.append(UInt8(truncatingIfNeeded: (crcValue >> 24) & 0xFF))
            let isize32 = UInt32(truncatingIfNeeded: isize)  // RFC 1952 § 2.3.1: ISIZE = input.count mod 2^32
            out.append(UInt8(truncatingIfNeeded: isize32 & 0xFF))
            out.append(UInt8(truncatingIfNeeded: (isize32 >> 8) & 0xFF))
            out.append(UInt8(truncatingIfNeeded: (isize32 >> 16) & 0xFF))
            out.append(UInt8(truncatingIfNeeded: (isize32 >> 24) & 0xFF))

            return Bytes(out)
        }

        // MARK: - Header

        private static func buildHeader(level: Level, filename: String?, modificationTime: UInt32) -> ContiguousArray<UInt8> {
            var out = ContiguousArray<UInt8>()
            out.reserveCapacity(11 + (filename?.utf8.count ?? 0))

            // RFC 1952 § 2.3.1 — 10-byte fixed header.
            out.append(0x1F)
            out.append(0x8B)
            out.append(0x08)  // CM = 8 (DEFLATE)
            let asciiFilename = filename.flatMap { Self.asAsciiCString($0) }
            let flg: UInt8 = (asciiFilename != nil) ? 0x08 : 0x00
            out.append(flg)
            out.append(UInt8(truncatingIfNeeded: modificationTime & 0xFF))
            out.append(UInt8(truncatingIfNeeded: (modificationTime >> 8) & 0xFF))
            out.append(UInt8(truncatingIfNeeded: (modificationTime >> 16) & 0xFF))
            out.append(UInt8(truncatingIfNeeded: (modificationTime >> 24) & 0xFF))
            let xfl: UInt8 = {
                switch level {
                case .best: return 2
                case .fast: return 4
                default: return 0
                }
            }()
            out.append(xfl)
            out.append(0xFF)  // OS = unknown

            if let bytes = asciiFilename {
                out.append(contentsOf: bytes)
                out.append(0x00)
            }
            return out
        }

        private static func asAsciiCString(_ s: String) -> [UInt8]? {
            var out: [UInt8] = []
            out.reserveCapacity(s.utf8.count)
            for scalar in s.unicodeScalars {
                let v = scalar.value
                if v == 0 || v > 0x7F { return nil }
                out.append(UInt8(v))
            }
            return out
        }
    }
}
```

## Test plan (~15 tests)

Round-trip:
1. Empty stream → decode = empty
2. Single chunk → decode = chunk
3. Two chunks → decode = concatenation
4. 100 tiny 1-byte chunks → round-trip
5. Large chunk (70 KiB) → round-trip
6. Mixed-size chunks → round-trip
7. Empty chunk in middle = no-op

Levels: `.none`, `.fast`, `.default`, `.best` — 4 round-trip tests.

Header/metadata:
12. Filename round-trip (FNAME flag set; bytes round-trip through Gzip.decode? — gzip decode discards FNAME so test that decoded payload matches input)
13. modificationTime round-trip (similar; tested via byte inspection of header)

Errors / edge:
14. Double-finish throws encoderFinished
15. update after finish silent no-op

## Stream-format

Output is `[10-byte header][optional FNAME + 0x00][DEFLATE body from Streaming.Encoder][4-byte CRC32 LE][4-byte ISIZE LE]`. Decodes via `Gzip.decode(_:)` v0.1+.

Empty-stream output: header (10 bytes) + DEFLATE-empty (5-byte stored-block terminator from `Deflate.Streaming.Encoder.finish()`) + 4-byte CRC32 = 0x00000000 LE + 4-byte ISIZE = 0x00000000 LE = 23 bytes total. Decodes to empty.

## Migration

Additive only — non-breaking. All v0.2 APIs unchanged. `Gzip.encode(_:)` continues to produce byte-equal output (v0.2 path untouched). `GzipError` gains 1 case (additive).

## Decisions locked

- New `Gzip.Streaming.Encoder` in parallel namespace. Existing `Gzip.Encoder` preserved verbatim.
- Per-chunk Deflate streaming via `Deflate.Streaming.Encoder` (inner state-bearing).
- Incremental CRC32 via `CRC.Digest<UInt32>(algorithm: .iso_hdlc)`.
- ISIZE counter as `UInt64` accumulator; truncated to UInt32 at trailer (RFC 1952 § 2.3.1 specifies low 32 bits modulo 2^32).
- Inner DeflateError wrapped as `GzipError.malformedDeflate` (defensive; double-finish guard prevents reaching it in practice).
- Sendable struct + State enum + canonical streaming-encoder shape.
- swift-deflate dep bumped 0.2.0 → 0.3.0 (for Streaming.Encoder access).

# swift-content-encoding v0.5 single-coding streaming — Design

**Phase 25 Tranche 25A** (RFC-0030). Existing package; additive non-breaking minor version bump.

**Date:** 2026-05-16

## Goal

Add `ContentEncoding.Streaming.Encoder(contentEncoding:, level:) throws` for single-coding HTTP `Content-Encoding` streaming. Dispatches to inner streaming encoders in swift-gzip / swift-zlib / swift-brotli per coding. Multi-coding chains throw `.multipleCodingsNotStreamable(header)` at init. Identity coding buffers internally and returns at `finish()`.

## Reality-check findings

- `ContentEncodingError` has 3 v0.4 cases: `.unsupportedEncoding(String)`, `.decodingFailed(String)`, `.encodingFailed(String)`. Need to add 2 cases: `.multipleCodingsNotStreamable(String)`, `.encoderFinished`.
- `ContentEncoding.parseCodings(_:)` is already `internal static` — reusable as-is.
- Existing exhaustive switch at `Tests/ContentEncodingTests/<file>:390` needs update.
- Package needs dep bumps: swift-gzip 0.2.0 → 0.3.0, swift-zlib 0.2.0 → 0.3.0, swift-brotli 0.2.0 → 0.3.0. swift-deflate stays at 0.2.0 (content-encoding doesn't import Deflate directly for `deflate` coding — it uses Zlib).
- All 4 inner streaming encoders share the canonical shape (init [throwing or not] / update / finish throwing). content-encoding wraps in an enum dispatch.
- `Brotli.Streaming.Encoder.init(quality:)` throws — but `.default` quality is always valid, so `try` in init never fires; wrap defensively as `.encodingFailed("br: ...")` if it ever does.
- The `level` parameter maps cleanly to gzip/zlib (both take `Deflate.Encoder.Level`). For brotli, `level` is ignored (uses brotli `.default` quality), matching v0.4 one-shot behavior.

**Conclusion:** Same canonical template as Phase 22-24. Estimate ~30-60 min wall-clock per the wrapper-package calibration refined in Gate 24.

## Scope

**In scope:**
- `ContentEncoding.Streaming` namespace enum.
- `ContentEncoding.Streaming.Encoder` struct: `init(contentEncoding:level:) throws` / `update(_:)` / `finish() throws -> Bytes`.
- Multi-coding chains throw `.multipleCodingsNotStreamable(header)` at init.
- Identity coding buffers into an accumulator; returns at finish.
- gzip/x-gzip/deflate/x-deflate/br codings dispatch to inner streaming encoders.
- 2 new `ContentEncodingError` cases.
- ~15-20 new tests (per-coding round-trip + multi-coding-throws + error cases + level coverage + edge cases).

**Out of scope (Phase 26+):**
- Multi-coding streaming chains (requires codec-tier `drain()` API).
- Streaming decode (no codec package has streaming decode yet).
- `reset()` for encoder reuse.
- Explicit flush API.
- `Level → brotli Quality` mapping (defer to v0.6 if demand).

**Anti-goals:**
- No changes to v0.4 `encode` / `decode` byte-for-byte output.
- No new dependencies beyond version bumps.
- No new core algorithms — pure orchestration over canonical streaming encoders.

## File changes

```
swift-content-encoding/
  Package.swift                                ← bump gzip/zlib/brotli 0.2 → 0.3
  Sources/ContentEncoding/
    ContentEncoding.swift                      ← +public enum Streaming namespace (~5 LOC)
    ContentEncodingError.swift                 ← +2 cases (~6 LOC)
    Streaming.swift                            ← NEW: Encoder struct + State + InnerEncoder enum (~120 LOC)
    Documentation.docc/ContentEncoding.md      ← Streaming topic group (~25 LOC)
  Tests/ContentEncodingTests/
    <existing>                                 ← extend exhaustive switch
    StreamingTests.swift                       ← NEW: ~18 tests (~270 LOC)
  CHANGELOG.md                                 ← v0.5.0 entry
  README.md                                    ← v0.5 install + streaming example + multi-coding limitation note
```

## Public API additions

```swift
extension ContentEncoding {
    public enum Streaming: Sendable {}
}

extension ContentEncoding.Streaming {
    public struct Encoder: Sendable {
        public typealias Level = Deflate.Encoder.Level

        public init(
            contentEncoding header: String,
            level: Level = .default
        ) throws(ContentEncodingError)

        public mutating func update(_ chunk: Bytes)
        public mutating func finish() throws(ContentEncodingError) -> Bytes
    }
}

public enum ContentEncodingError: Error, Equatable, Sendable {
    // ... existing 3 cases ...

    /// Streaming encoder: header carried multiple codings (e.g. "gzip, br").
    /// v0.5 streaming supports single-coding only. Fall back to
    /// ``ContentEncoding/encode(_:contentEncoding:level:)`` one-shot for
    /// multi-coding chains.
    case multipleCodingsNotStreamable(String)

    /// Encoder: ``ContentEncoding/Streaming/Encoder/finish()`` was called
    /// twice on the same encoder.
    case encoderFinished
}
```

## Internals

`Sources/ContentEncoding/Streaming.swift`:

```swift
import Bytes
import Brotli
import Deflate
import Gzip
import Zlib

extension ContentEncoding.Streaming {
    public struct Encoder: Sendable {
        public typealias Level = Deflate.Encoder.Level

        private enum State: Sendable {
            case open
            case finished
        }

        private enum InnerEncoder: Sendable {
            /// Identity: passthrough; bytes accumulate in `accumulator` and
            /// are returned verbatim at `finish()`.
            case identity(Bytes)
            case gzip(Gzip.Streaming.Encoder)
            case deflate(Zlib.Streaming.Encoder)   // RFC 7230 § 4.2.2 — "deflate" = zlib-framed
            case brotli(Brotli.Streaming.Encoder)
        }

        public let header: String
        public let level: Level

        private var inner: InnerEncoder
        private var state: State

        public init(
            contentEncoding header: String,
            level: Level = .default
        ) throws(ContentEncodingError) {
            self.header = header
            self.level = level

            let codings = ContentEncoding.parseCodings(header)
            if codings.isEmpty {
                // Empty / whitespace header → identity passthrough.
                self.inner = .identity(Bytes())
                self.state = .open
                return
            }
            if codings.count > 1 {
                throw .multipleCodingsNotStreamable(header)
            }
            let coding = codings[0]
            switch coding {
            case "identity":
                self.inner = .identity(Bytes())
            case "gzip", "x-gzip":
                self.inner = .gzip(Gzip.Streaming.Encoder(level: level))
            case "deflate", "x-deflate":
                self.inner = .deflate(Zlib.Streaming.Encoder(level: level))
            case "br":
                // Brotli.Streaming.Encoder(quality: .default) only throws on
                // invalid quality. `.default` is always valid. Wrap defensively.
                do {
                    self.inner = .brotli(try Brotli.Streaming.Encoder(quality: .default))
                } catch {
                    throw .encodingFailed("br: \(error)")
                }
            default:
                throw .unsupportedEncoding(coding)
            }
            self.state = .open
        }

        public mutating func update(_ chunk: Bytes) {
            guard case .open = state else { return }
            if chunk.isEmpty { return }
            switch inner {
            case .identity(var acc):
                acc.append(contentsOf: chunk.storage)
                inner = .identity(acc)
            case .gzip(var enc):
                enc.update(chunk)
                inner = .gzip(enc)
            case .deflate(var enc):
                enc.update(chunk)
                inner = .deflate(enc)
            case .brotli(var enc):
                enc.update(chunk)
                inner = .brotli(enc)
            }
        }

        public mutating func finish() throws(ContentEncodingError) -> Bytes {
            guard case .open = state else { throw .encoderFinished }
            state = .finished

            switch inner {
            case .identity(let acc):
                return acc
            case .gzip(var enc):
                do {
                    return try enc.finish()
                } catch {
                    throw .encodingFailed("gzip: \(error)")
                }
            case .deflate(var enc):
                do {
                    return try enc.finish()
                } catch {
                    throw .encodingFailed("deflate: \(error)")
                }
            case .brotli(var enc):
                do {
                    return try enc.finish()
                } catch {
                    throw .encodingFailed("br: \(error)")
                }
            }
        }
    }
}
```

## Test plan (~18 tests)

Round-trip per coding (v0.5 streaming encode → v0.4 decode):

1. identity → empty header → passthrough
2. identity → explicit "identity"
3. gzip
4. x-gzip
5. deflate
6. x-deflate
7. br

Multi-chunk + boundary:

8. Two chunks → concatenation
9. Empty chunk in middle = no-op
10. Single-byte stream

Levels:

11. `.fast` gzip
12. `.best` deflate

Errors:

13. Multi-coding "gzip, br" throws `.multipleCodingsNotStreamable`
14. Multi-coding "deflate, gzip" throws same
15. Unsupported coding "zstd" throws `.unsupportedEncoding`
16. Double-finish throws `.encoderFinished`
17. Update after finish is silent no-op (then double-finish throws)

Header parsing:

18. Whitespace-tolerant: `"  gzip  "` accepted

## Migration

Additive only — non-breaking. v0.4 APIs unchanged.

## Decisions locked

- New `.multipleCodingsNotStreamable(String)` case (clearer semantics than overloading `.unsupportedEncoding`).
- New `.encoderFinished` case.
- Identity coding buffers internally; returns accumulator at finish.
- Brotli streaming uses `.default` quality always (matches v0.4 br-encode behavior). `level` parameter ignored for br.
- Empty / whitespace header → identity passthrough (not throw).
- Inner state via enum with associated-value streaming encoders.
- swift-gzip / swift-zlib / swift-brotli deps bumped 0.2.0 → 0.3.0.
- swift-deflate dep unchanged (not directly used).
- Sendable struct + State enum + canonical streaming-encoder shape.

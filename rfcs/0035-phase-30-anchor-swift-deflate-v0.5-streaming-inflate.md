# RFC-0035 — Phase 30 anchor: swift-deflate v0.5 (streaming inflate)

| Field | Value |
| --- | --- |
| Status | Accepted |
| Author | bare-swift project lead |
| Created | 2026-05-17 |
| Resolution | Accepted 2026-05-17 — Phase 30 plans may begin authoring. Single existing package: swift-deflate v0.5. |

## Summary

Anchor Phase 30 on **swift-deflate v0.5** — additive non-breaking minor version bump adding `Deflate.Streaming.Decoder` (init/update/finish) for streaming RFC 1951 INFLATE. Opens the streaming-decode side of the codec tier; bottom of the codec stack (gzip/zlib wrap deflate). Single-tranche; estimated ~3-5 hours (state-machine work, NOT wrapper-pattern). Per-tranche estimate **deferred to brainstorm reality-check** per Gate 22 canonical pattern.

## Problem

[Gate 29 retrospective](../docs/gates/2026-05-17-gate-29-retrospective.md) item 5 requires "Phase 30 anchor decision recorded as an RFC" before Phase 30 plans begin. The retrospective surveyed eleven candidate waves; this RFC formalizes the choice.

**Concrete gap:** swift-deflate v0.1 (Phase 7B, shipped 2026-05-10) ships a one-shot inflate: `Deflate.inflate(_ compressed: Bytes) throws(DeflateError) -> Bytes` that consumes a complete `Bytes` payload and returns the full decompressed output. There is no streaming-decode API.

Phase 22-28 shipped streaming-encode across all 4 codec packages (brotli, deflate, gzip, zlib) plus the HTTP-layer multi-coding pipeline via swift-content-encoding v0.6. **Streaming-decode is the symmetric remaining work.** Adopter use cases for streaming decode include:

- Large HTTP response bodies (stream-decode chunk-by-chunk without buffering full body).
- Server-sent events with compressed payloads.
- Long-running WebSocket streams.
- Log-tailing over HTTP.

**Why now (Phase 30):**
1. Streaming-encode is feature-complete. Streaming-decode is the natural symmetric extension.
2. swift-deflate is the **bottom of the codec stack** — gzip/zlib wrap deflate. Deflate v0.5 unblocks gzip + zlib streaming decode in Phase 31+.
3. Modest single-tranche scope (~3-5hr); state-machine work but bounded.
4. Audience continuity with Phase 22-28 codec streaming sweep.

**Why not the alternatives (full survey in Gate 29 retro § Item 5):**
- **Full 4-tranche streaming decoders sweep** — too heavy for one phase. Spread across Phase 30/31/32.
- **swift-brotli v0.5 first** — independent of deflate; could be Phase 31 alternative. swift-deflate first preferred (downstream multi-package leverage).
- **swift-brotli v0.5 + swift-deflate v0.5 window carry** — ratio improvement; not blocking. Defer.
- **swift-oauth2-client v0.4** — v0.3 just shipped; no settle time.
- **B3 + Jaeger, idna v0.3** — no demand.
- **RS-family JWT** — STILL BLOCKED on swift-crypto `_RSA` SPI.
- **Package rename** — premature.

## Proposal

### Anchor: swift-deflate v0.5 (single existing package, additive minor bump)

**swift-deflate v0.5** — add `Deflate.Streaming.Decoder` alongside v0.1's `Deflate.inflate(_:)` one-shot. v0.1 inflate + v0.2 one-shot encode + v0.3 streaming encode + v0.4 drain preserved verbatim.

| Addition | Description |
|---|---|
| `Deflate.Streaming.Decoder` struct | Sendable value-type streaming decoder. Mirrors `Streaming.Encoder` shape: init / update(_:) / finish() throws -> Bytes. |
| `init()` | No parameters (no level / quality for decoder). Initializes empty state. |
| `mutating func update(_ chunk: Bytes)` | Feed compressed input bytes. Decoder buffers internal state and emits any newly-decodable output to an internal buffer. Empty chunk = no-op. No-op after finish. |
| `mutating func finish() throws(DeflateError) -> Bytes` | Validate stream terminator was seen; return all accumulated decompressed output. Throws `decoderFinished` on double-call, existing inflate errors (truncated, invalidHuffmanTable, etc.) on malformed input. |
| Possibly: `var output: Bytes { get }` or `mutating func drain() -> Bytes` | TBD by brainstorm — whether to expose incremental output mid-stream. Likely YES (matches streaming-encode side's drain API). |
| Likely new error case | `decoderFinished` for double-finish. |

### Key API shape (subject to brainstorm refinement)

```swift
// v0.5 streaming usage (proposed):
import Deflate
import Bytes

var decoder = Deflate.Streaming.Decoder()
decoder.update(compressedChunk1)
decoder.update(compressedChunk2)
let decompressed = try decoder.finish()
// `decompressed` == Deflate.inflate(compressedChunk1 + compressedChunk2)
```

If brainstorm decides to expose incremental output:

```swift
var decoder = Deflate.Streaming.Decoder()
decoder.update(compressedChunk1)
let partial1 = decoder.drain()  // bytes decoded so far
decoder.update(compressedChunk2)
let partial2 = decoder.drain()
let final = try decoder.finish()
// total decoded == partial1 + partial2 + final
```

### Decomposition

**Shape A (recommended pending reality-check): single tranche, full streaming inflate.**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 30A | swift-deflate v0.5 | `Deflate.Streaming.Decoder` (init/update/finish + possibly drain) + Inflater state-machine refactor for yield-and-resume + ~12-18 tests | ~3-5 hours (pending reality-check) |

**Shape B (alternate): two tranches if state-machine refactor exceeds scope.**

| Tranche | Package | Scope | Estimated calendar |
|---|---|---|---|
| 30A | swift-deflate v0.5 | Streaming inflate for stored + fixed-Huffman blocks (simpler state machines) | ~2-3 hours |
| 30B | swift-deflate v0.6 | Streaming inflate for dynamic-Huffman blocks (more complex state) | ~2-3 hours |

Decision deferred to brainstorm phase. Shape A is the working assumption; Shape B fallback only if dynamic-Huffman state-machine refactor exceeds budget.

### Critical brainstorm questions (lock at brainstorm time)

1. **Output buffering vs incremental yield.** Does `update(_:)` emit decoded bytes immediately (caller pulls via `drain()`)? Or does `finish()` return all output at once? **Likely:** matches the streaming-encode side — internal output buffer; `finish()` returns accumulated bytes; possibly `drain()` for mid-stream pull. Symmetric with v0.4's encoder drain.
2. **State-machine yield points.** Where can decoder safely pause when input is exhausted? Bit-level vs byte-level vs symbol-level? **Likely:** bit-level via BitReader save/restore semantics.
3. **Input buffering.** Does the decoder buffer pending input chunks internally? Or does it require the caller to provide enough input per `update`? **Likely:** decoder buffers internally (matches caller-friendly streaming-encode API).
4. **Error semantics for incomplete input at finish.** What does `finish()` do if the compressed stream was truncated mid-block? **Likely:** throws `.truncated` (existing v0.1 error case).
5. **`drain()` API surface.** TBD whether to expose incremental output. **Pros:** symmetric with encoder side; useful for large streams. **Cons:** API surface area; harder to test (state interaction).

### Test surface (~12-18 tests)

| Test category | Scope |
|---|---|
| Round-trip via v0.2 one-shot encoder | Encode "hello", encode "hello world", encode pangram, encode 70 KiB random → stream-decode each → bytes match | 4-5 tests |
| Multi-chunk input | Split a one-shot-encoded output into 2/3/many chunks → stream-decode → bytes match | 3-4 tests |
| Block-type coverage | Stream-decode stored / fixed-Huffman / dynamic-Huffman block-type outputs from v0.2 encoder | 3 tests |
| Error cases | Truncated input throws `.truncated`; double-finish throws `.decoderFinished`; update-after-finish silent no-op | 3 tests |
| Edge cases | Empty stream decodes to empty; single-byte input chunks (many tiny updates); single-byte output | 3 tests |

Estimated 12-18 new tests. Total package tests post-30A: 73 v0.4 + 15 = ~88.

### Acceptance criteria

- v0.5.0 ships with `Deflate.Streaming.Decoder` (init/update/finish + possibly drain).
- v0.1 + v0.2 + v0.3 + v0.4 APIs unchanged. `Deflate.inflate(_:)` continues byte-for-byte unchanged (regression-tested).
- Stream-decoded output equals one-shot `Deflate.inflate(_:)` output for all test cases.
- CI green on macOS + Linux, first try.
- DocC includes a `### Streaming inflate (v0.5+)` topic group.
- CHANGELOG + README updated.

### Out of scope (deferred to v0.6+)

- **`reset()` for decoder reuse.** Symmetric to encoder side; v0.6+ candidate.
- **Async/AsyncSequence wrapper.** Caller composes with their own async runtime.
- **`Decoder.drain()`** if not in scope per brainstorm.
- **Window carry across chunks** (decoder side has implicit window already; encoder side is the deferred work).

### Migration (v0.4 → v0.5)

**Additive only — non-breaking.** All v0.4 APIs unchanged:
- `Deflate.inflate(_:)` continues byte-for-byte (regression-tested).
- `Deflate.encode(_:level:)` unchanged.
- `Deflate.Streaming.Encoder` + drain() unchanged.
- `DeflateError` adds 1 new case (additive).

### Risk

**Medium-high. First state-machine refactor in the bare-swift codec tier.**

| Risk | Mitigation |
|---|---|
| Inflater state machine doesn't cleanly split at yield points | Brainstorm reads v0.1 Inflater.swift carefully; settles bit-level vs byte-level yield. |
| Dynamic-Huffman block decode requires reading prefix codes mid-stream; yield-and-resume across prefix-code-read might be tricky | Save/restore BitReader state + accumulated symbols. Possibly defer dynamic-Huffman streaming to Shape B tranche if too complex. |
| Output buffer grows unbounded if `drain()` not exposed and caller streams large input | Document the trade-off. v0.6 could add bounded buffer + backpressure. |
| Streaming inflate output may differ from one-shot inflate byte-for-byte due to internal buffering | Streaming inflate MUST produce byte-identical output to one-shot — tested via round-trip. |
| Test setup: how to construct compressed input for streaming-decode tests? | Use v0.2 one-shot encoder to generate compressed bytes; split + stream-decode. |

## Alternatives considered

See [Gate 29 retrospective](../docs/gates/2026-05-17-gate-29-retrospective.md) § Item 5 for the full candidate matrix.

## Dependencies

- No new direct dependencies on swift-deflate.
- Phase 31 will bump swift-gzip + swift-zlib deps to 0.5+ for streaming-inflate access.

## References

- [RFC 1951](https://www.rfc-editor.org/rfc/rfc1951) — DEFLATE compressed data format.
- [Gate 29 retrospective](../docs/gates/2026-05-17-gate-29-retrospective.md) — anchor decision rationale.
- [RFC-0028](0028-phase-23-anchor-swift-deflate-v0.3-streaming-encoder.md) — Phase 23 anchor (deflate v0.3 streaming encoder); the canonical Streaming.Encoder shape that Streaming.Decoder mirrors.
- swift-deflate v0.4 CHANGELOG — current state (one-shot inflate + streaming encode + drain).

## Decision

**Accepted 2026-05-17.** Phase 30 anchored on **swift-deflate v0.5 streaming inflate**. Brainstorm + plan + execute via the bare-swift inline-execution pattern. Per-tranche calendar estimate **deferred to brainstorm reality-check** per Gate 22's canonical pattern (~3-5 hours expected; state-machine-work bracket).

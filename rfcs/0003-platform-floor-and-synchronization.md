# RFC-0003 — Platform floor & `Synchronization` primitives

| Field | Value |
| --- | --- |
| Status | Draft |
| Author | bare-swift project lead |
| Created | 2026-05-07 |
| Resolution | _(filled at merge)_ |

## Summary

Allow a bare-swift package to declare `.macOS(.v15)` instead of the org-wide `.macOS(.v14)` floor when the package legitimately needs Swift 6's `Synchronization.Atomic` or `Synchronization.Mutex` primitives. The org-wide default stays at macOS 14; per-package overrides are documented in CHANGELOG and explained in the package README.

## Problem

RFC-0001 §"Swift target and platforms" mandates `macOS 14+` for every package. swift-prometheus (Tranche 1C) needed `Synchronization.Atomic<UInt64>` for wait-free counter increments — the canonical primitive for hot-path metric updates. `Synchronization.Atomic` is an **availability: macOS 15+** API on Apple platforms (the type ships in Swift 6.0's standard library, but its operations are gated on macOS 15 / iOS 18 / watchOS 11 / tvOS 18 because the underlying `Builtin.Atomic*` calls require those platform versions).

Phase 1 result: swift-prometheus is the only Phase 1 package that targets macOS 15. The other nine target macOS 14 cleanly.

The question is not "should Synchronization be allowed?" — it already shipped, and rolling it back would mean either:

- Writing a Mutex around `Double` (turning lock-free atomic increments into a serialized write), or
- Taking a dependency on `swift-atomics` (https://github.com/apple/swift-atomics), which adds a dependency footprint we've avoided in Phase 1.

The question is whether **the org-wide platform floor stays at 14** with per-package override (this RFC), or **bumps uniformly to 15** so every package can use Synchronization without ceremony.

## Proposal

### Org-wide platform floor: macOS 14 (unchanged from RFC-0001)

The default `.macOS(.v14)` declared by `bare-swift new` does not change. Most Phase 1 packages — and most expected Phase 2 packages — don't need any Synchronization primitive and shouldn't pay the macOS-14-vs-15 audience cost.

### Per-package override: macOS 15 with documented justification

A package may declare `.macOS(.v15)` in its `Package.swift` if and only if:

1. It uses `import Synchronization` (`Atomic` or `Mutex`), **and**
2. The README's "Platform" section names macOS 15+ as the floor and explains why, **and**
3. The CHANGELOG's v0.1 entry calls out the platform floor explicitly under a `### Platform` heading.

No other reason qualifies as a per-package override under this RFC. (If a future package needs macOS 15 for a *different* reason — e.g., a stdlib API only available on macOS 15 — that's an RFC amendment, not a unilateral choice.)

### What about Linux?

`platforms:` in `Package.swift` only affects Apple platforms. Linux availability is governed by the Swift toolchain version (`swift-tools-version: 6.0`). `Synchronization.Atomic` and `Synchronization.Mutex` are available on Linux Swift 6.0+ without per-platform availability gating. swift-prometheus's CI confirms this — Linux Swift 6.0 and 6.1 both build and test cleanly.

### What about swift-atomics as an alternative dependency?

Rejected for Phase 2's expected scope. swift-atomics is a well-engineered Apple package that backports `Atomic` to older platforms via `import Atomics`. Pulling it in adds a package dependency and an `Atomics` import in our user-visible compile graph. For a metrics-style hot path, the marginal portability gain (macOS 14 instead of 15) does not justify the dependency surface. Future packages with stricter portability needs may revisit this on a case-by-case basis with another RFC.

### Alternatives considered

**Bump the org-wide floor to macOS 15.** Rejected: penalizes the nine Phase 1 packages that don't need Synchronization. macOS 14 still has meaningful adopter share (especially server-side, where macOS-as-deployment-target matters less but macOS-as-dev-target matters for the CI matrix). Holding the floor at 14 keeps the on-ramp wide.

**Forbid `Synchronization` entirely; mandate swift-atomics.** Rejected: see above. swift-atomics adds a dependency we don't need org-wide.

**Write a Mutex-protected wrapper instead of `Atomic`.** Rejected for swift-prometheus: turns lock-free counter increments into serialized writes. For a metrics library, that's a load-bearing performance regression on the hottest call path.

## Drawbacks

- Adopters running macOS 14 cannot use swift-prometheus. Documented in its README and CHANGELOG.
- Per-package floor variation makes "what does bare-swift require?" harder to answer in one sentence. The umbrella site should list each package's floor in its index entry.

## Unresolved questions

- If swift-atomics ships a meaningful improvement (e.g., better optimization, smaller binary), should we revisit and standardize on it? Out of scope for this RFC; revisit at Gate 2 or earlier if the situation changes.

## Migration impact

- swift-prometheus already ships with `.macOS(.v15)` and the platform-justification CHANGELOG entry. Its v0.1.0 release predates this RFC; the RFC retroactively codifies its reality.
- `bare-swift new` continues to scaffold `.macOS(.v14)` by default. The package author bumps to `.v15` in the same commit that adds `import Synchronization`, with the CHANGELOG entry and README note.
- Umbrella package index (`bare-swift/packages/index.json`) currently does not record per-package platform floors. **Follow-up:** add a `platforms` field to the index schema; surface "macOS 15+" inline on the umbrella site for affected packages. Tracked separately, not blocking acceptance of this RFC.

## Resolution

_(Filled by the project lead at merge time.)_

- **Accepted** / **Rejected** / **Withdrawn**
- Date:
- Notes:

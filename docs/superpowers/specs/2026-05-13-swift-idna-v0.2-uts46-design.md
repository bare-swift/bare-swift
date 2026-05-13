# swift-idna v0.2 — UTS #46 + NFC Design

**Phase 15 Tranche 15A** (RFC-0020). Plus optional 15B: swift-uri v0.3 auto-encode.

**Date:** 2026-05-13

---

## Goal

Upgrade swift-idna from its v0.1 Punycode-only codec to a full UTS #46 mapping + NFC normalization pipeline so callers no longer need to pre-normalize. Closes Phase 13's documented deferral and unblocks swift-uri v0.3 auto-encode.

## Scope (Shape B — honest-scope v0.2 + 15B stretch)

**In scope for v0.2:**
- IdnaMappingTable (UTS #46 § 4 Table 1; Unicode 16.0)
- NFC normalization (Unicode TR #15)
- UTS #46 ToASCII pipeline (map → NFC → punycode-encode → validate)
- UTS #46 ToUnicode pipeline (punycode-decode → map → NFC → validate)
- Updated `IDNA.toASCII(_:)` signature: now `throws(IDNAError)`
- New `IDNAError` cases: `disallowedCharacter(Unicode.Scalar)`, `mappingNotFound(Unicode.Scalar)`

**Out of scope (deferred to v0.3):**
- Bidi rule (RFC 5893)
- ContextJ rule (RFC 5892 + RFC 5891 § 4.2.3)
- ContextO rule
- Transitional handling toggle (hardcode nontransitional, the WHATWG default)
- STD3 ASCII strict toggle (hardcode non-strict)
- Custom mapping tables (bundle Unicode 16.0 only)

**Anti-goals:**
- No Foundation in `Sources/IDNA` or test target (per [[feedback-no-foundation-in-tests]]).
- No `Bundle.module` resource loading; embed data as Swift static literals (swift-publicsuffix v0.1 pattern).
- No breaking change to the existing Punycode codec API.

**Stretch (15B): swift-uri v0.3**
- swift-uri auto-encodes Unicode hostnames at `URI(string:)` parse time.
- `URIError.idnaNotSupported` is replaced by `URIError.idnaProcessing(IDNAError)` (breaking; minor version bump).

---

## Module structure

```
Sources/IDNA/
  Documentation.docc/
    IDNA.md                      (updated)
  Generated/
    IdnaMappingTable+Data.swift  NEW — generated UTS #46 mapping (~50 KiB)
    NFC+Data.swift               NEW — generated CCC + decomp + composition (~70 KiB)
  IDNA.swift                     UPDATED — toASCII becomes throwing
  IDNAError.swift                UPDATED — extended with 2 new cases
  Punycode.swift                 unchanged
  PunycodeCodec.swift            unchanged
  IdnaMappingTable.swift         NEW — range-table lookup (~80 LOC)
  NFC.swift                      NEW — decompose → reorder → compose (~250 LOC)
  UTS46.swift                    NEW — ToASCII / ToUnicode pipelines (~180 LOC)
Tests/IDNATests/
  IDNATests.swift                unchanged (38 v0.1 tests must still pass)
  IdnaMappingTableTests.swift    NEW (8 tests)
  NFCTests.swift                 NEW (15 tests)
  UTS46ToASCIITests.swift        NEW (25 tests)
  UTS46ToUnicodeTests.swift      NEW (25 tests)
  RoundTripTests.swift           NEW (20 tests, parametrized)
  ErrorCaseTests.swift           NEW (6 tests)
scripts/
  regenerate-idna-tables.swift   NEW — Foundation-y host script (~200 LOC)
```

---

## Data sources (Unicode 16.0)

| Source | URL | Purpose |
|---|---|---|
| IdnaMappingTable.txt | `https://www.unicode.org/Public/idna/16.0.0/IdnaMappingTable.txt` | UTS #46 mapping ranges |
| UnicodeData.txt | `https://www.unicode.org/Public/16.0.0/ucd/UnicodeData.txt` | Canonical_Combining_Class + Canonical_Decomposition |
| CompositionExclusions.txt | `https://www.unicode.org/Public/16.0.0/ucd/CompositionExclusions.txt` | NFC composition exclusion set |
| DerivedNormalizationProps.txt | `https://www.unicode.org/Public/16.0.0/ucd/DerivedNormalizationProps.txt` | NFC_Quick_Check property (optional fast path) |

**License:** Unicode terms of use (no copyright; license is in `LICENSE-UNICODE` to be added). NOTICE must credit Unicode Inc.

---

## Public API

```swift
public enum IDNA: Sendable {
    /// Convert Unicode hostname to ACE per UTS #46.
    /// BREAKING from v0.1: now throws.
    public static func toASCII(_ host: String) throws(IDNAError) -> String

    /// Convert ACE hostname back to Unicode per UTS #46.
    public static func toUnicode(_ host: String) throws(IDNAError) -> String

    /// Case-insensitive ACE-form equality. Catches IDNAError → returns false.
    public static func compareEqual(_ a: String, _ b: String) -> Bool
}

public enum IDNAError: Error, Equatable, Sendable {
    // v0.1 cases — unchanged
    case labelTooLong
    case invalidPunycode(String)
    case invalidCodePoint
    case invalidHyphenPosition
    // v0.2 — new
    case disallowedCharacter(Unicode.Scalar)
    case mappingNotFound(Unicode.Scalar)
}
```

`compareEqual` stays non-throwing — equality queries shouldn't trap.

---

## Algorithm sketches

### IdnaMappingTable lookup

**Internal status enum** (not public):
```swift
internal enum Status: UInt8 {
    case valid = 0
    case ignored = 1
    case mapped = 2                      // mappingOffset points to replacement
    case deviation = 3                   // treat as valid (nontransitional)
    case disallowed = 4
    case disallowed_STD3_valid = 5       // treat as valid (non-strict default)
    case disallowed_STD3_mapped = 6      // treat as mapped (non-strict default)
}
```

**Table layout:** sorted array of `(startCP: UInt32, endCP: UInt32, status: UInt8, mappingOffset: UInt32)`. Binary search by code point.

**Mapping data:** separate `[UInt32]` flat buffer; `mappingOffset` points to a length-prefixed run of replacement scalars.

### NFC

**Inputs:** `[Unicode.Scalar]`. **Output:** `[Unicode.Scalar]`. No Foundation.

**Tables:**
- `CCC: [(startCP: UInt32, endCP: UInt32, ccc: UInt8)]` — sorted, binary-searchable.
- `Decomposition: [UInt32: (offset: UInt32, length: UInt8)]` — packed.
- `Composition: [(first: UInt32, second: UInt32, result: UInt32)]` — sorted by (first, second).
- `NFC_QC_No: Set<UInt32>` — quick check.

**Algorithm (Unicode TR #15):**
1. **Canonical Decomposition:** for each scalar, recursively expand via Canonical_Decomposition until reaching scalars with no decomposition.
2. **Canonical Ordering:** stable sort runs of combining marks (CCC > 0) by CCC value.
3. **Canonical Composition:** scan; if (starter, mark) ∈ composition table AND not in exclusion set, replace pair with composite.
4. Output the resulting scalar array.

**Optimization:** if all input scalars pass `NFC_QC == Yes` AND have CCC = 0, skip the algorithm (fast path).

### UTS #46 ToASCII (§ 4.1)

```
1. domain_scalars = []
   for scalar in input:
       status, mapping = IdnaMappingTable.lookup(scalar)
       switch status:
         case .valid, .deviation, .disallowed_STD3_valid:
             domain_scalars.append(scalar)
         case .mapped, .disallowed_STD3_mapped:
             domain_scalars.append(contentsOf: mapping)
         case .ignored:
             continue
         case .disallowed:
             throw .disallowedCharacter(scalar)
2. normalized = NFC.normalize(domain_scalars)
3. labels = split(normalized, on: ".")
4. ace_labels = []
   for label in labels:
       if label.allSatisfy({ $0.value < 0x80 }):
           ace_labels.append(label)
       else:
           ace_labels.append("xn--" + Punycode.encode(label))
       validate(ace_labels.last)   // existing v0.1 RFC 5891 § 4.2 checks
5. return ace_labels.joined(separator: ".")
```

### UTS #46 ToUnicode

```
1. labels = split(input, on: ".")
2. decoded_labels = []
   for label in labels:
       if label.hasPrefix("xn--"):
           decoded_labels.append(try Punycode.decode(label.drop("xn--")))
       else:
           decoded_labels.append(label)
3. mapped_scalars = []
   for scalar in decoded_labels.joined("."):
       (status, mapping) = IdnaMappingTable.lookup(scalar)
       // same mapping rules as ToASCII
4. normalized = NFC.normalize(mapped_scalars)
5. validate label lengths + hyphen rules (existing v0.1 checks)
6. return normalized as String
```

---

## CI considerations

**Sanitizers OFF for the test target.** v0.2 embeds ~120 KiB of static data; per Gate 10's lesson, sanitizers ON over >100 KiB data triggers >30-min CI timeouts. CI workflow YAML needs `-Xswiftc -DSANITIZERS_OFF` or equivalent for the test target (matrix entry). Reference: how swift-publicsuffix's `.github/workflows/ci.yml` handles its 140 KiB PSL data.

**Test target Foundation-free.** Test vectors embedded as Swift literals; no `Foundation.URL`, no `Bundle`, no file I/O.

**Tag-triggered CI may be slow.** Initial compile of the Generated/ files is heavy (~500 LOC of dense literal data). Expect ~15-min first build per CI matrix entry. Subsequent compiles with no Generated/ changes are fast (cached).

---

## Test plan

Total: **~99 new tests**, on top of the 38 v0.1 tests (which must still pass).

| Suite | New tests | What's covered |
|---|---|---|
| `IdnaMappingTableTests` | 8 | valid / ignored / mapped / deviation / disallowed / disallowed_STD3_* / boundary code points |
| `NFCTests` | 15 | NormalizationTest.txt curated subset (composed/decomposed pairs, Hangul, combining marks, identity cases) |
| `UTS46ToASCIITests` | 25 | IdnaTestV2.txt curated subset: mixed-case Unicode, full-width ASCII, ligatures, deviation chars |
| `UTS46ToUnicodeTests` | 25 | Same vectors reversed |
| `RoundTripTests` | 20 | Parametrized over 10 well-known IDN examples × encode+decode |
| `ErrorCaseTests` | 6 | `disallowedCharacter` for known disallowed scalars; `mappingNotFound` is defensive |

All 38 v0.1 tests (Punycode encode/decode, label-level API, validity) continue to pass unchanged.

---

## LOC budget

| Component | LOC |
|---|---|
| `IdnaMappingTable.swift` (lookup wrapper) | ~80 |
| `NFC.swift` (algorithm) | ~250 |
| `UTS46.swift` (pipelines) | ~180 |
| `IDNA.swift` (updated API) | ~120 |
| `IDNAError.swift` (extended) | ~50 |
| `scripts/regenerate-idna-tables.swift` | ~200 |
| Tests (combined) | ~700 |
| **Hand-written total** | **~1580** |
| Generated data files | ~500 (data; not "real" LOC) |

Embedded bundle size: ~120 KiB packed.

---

## Migration (v0.1 → v0.2)

CHANGELOG migration section:

```markdown
### Migration (v0.1 → v0.2)

- **BREAKING: `IDNA.toASCII(_:)` is now `throws(IDNAError)`.** Callers that previously assumed it couldn't fail must now handle errors.
  - Old: `let ace = IDNA.toASCII(host)`
  - New: `let ace = try IDNA.toASCII(host)`
- **Behavior change: no more pre-normalization required.** v0.1 expected callers to lowercase + NFC-normalize first. v0.2 does both internally per UTS #46.
- Existing Punycode-level API (`IDNA.Punycode.encode/decode/encodeLabel/decodeLabel`) is unchanged.
- `IDNA.toUnicode(_:)` signature unchanged; behavior is now UTS #46-conformant.
- `IDNA.compareEqual(_:_:)` signature and non-throwing contract unchanged.
```

---

## 15B (stretch): swift-uri v0.3

After 15A ships:

**Changes:**
- `swift-uri` bumps `0.2.0 → 0.3.0`
- `Package.swift` adds dep: swift-idna 0.2.0
- `URI(string:)` parser: when host portion contains non-ASCII, call `try IDNA.toASCII(_:)`. On success, store ACE form in `URI.Host`. On `IDNAError`, throw `URIError.idnaProcessing(IDNAError)`.
- `URIError` enum: replace `case idnaNotSupported` with `case idnaProcessing(IDNAError)`. **Breaking; minor version bump.**

**Test plan:**
- ~10 new tests in swift-uri exercising Unicode hostname auto-encode.
- All 200+ existing swift-uri v0.1/v0.2 tests must continue to pass (the `idnaNotSupported` tests get updated to expect the new error case).

**LOC:** ~50 LOC additive in swift-uri.

**CHANGELOG migration:**
```markdown
- BREAKING: `URIError.idnaNotSupported` is replaced by `URIError.idnaProcessing(IDNAError)`.
- BREAKING in behavior: Unicode hostnames now parse end-to-end (previously threw `.idnaNotSupported`).
```

---

## Ship sequencing

1. **15A ship cycle:** swift-idna v0.2 — generate tables, write algorithm, tests, push, watch CI, tag, watch release, bump umbrella.
2. **15B ship cycle:** swift-uri v0.3 — add dep, wire parser, tests, ship.
3. **Memory closure:** `project_phase15_15a_shipped.md`, `project_phase15_15b_shipped.md`, MEMORY.md updates.

Total Phase 15 calendar: ~2 days wall-clock (1.5 day 15A + 0.5 day 15B).

---

## Decisions log (locked)

- **Shape B** (defer Bidi+ContextJ to v0.3, include 15B stretch). Selected via AskUserQuestion.
- Embed data as Swift static literals; no Foundation in `Sources/IDNA`.
- `IDNA.toASCII(_:)` becomes throwing (breaking minor-version change; v0.x SemVer permits).
- `IDNAError` extends with 2 new cases (Bidi/ContextJ cases deferred until v0.3 lands those features).
- Nontransitional and non-STD3 hardcoded; no toggles.
- Generation script lives in `scripts/`, may use Foundation (host-side tool, not in the package).
- Test target stays Foundation-free; vectors as Swift literals.
- Sanitizers OFF for v0.2 test target (Gate 10 lesson).

# bare-swift RFCs

This directory holds RFCs (Request for Comments) for cross-package design decisions in the bare-swift ecosystem.

## When to write an RFC

- Any cross-package decision (naming, error model, Sendable conventions).
- Any 1.0 API freeze on a package.
- Any new convention.
- Any package whose scope is contentious.

## When NOT to write an RFC

- Internal package design.
- Bug fixes.
- Additive non-breaking changes within an existing package.

## Process

1. Copy [`template.md`](./template.md) to `<NNNN>-<slug>.md` (next sequential number).
2. Open a PR with the draft RFC.
3. Discussion runs for at least 7 days.
4. Project lead records the resolution in the "Resolution" section and merges (accepted) or closes (rejected) the PR.

Numbers are assigned at merge time to avoid collisions; reserve a number by commenting on your PR.

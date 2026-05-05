# bare-swift — manual setup checklist

Phase 0 produces files locally. The following steps require human action on GitHub and other external services. Walk through them in order.

## Org

- [ ] Create the GitHub organization `bare-swift`.
- [ ] Add the project lead's account as the org owner.
- [ ] Set the org's public-facing email and homepage to point at this README / the future site.

## Repos to create on GitHub

- [ ] `bare-swift/bare-swift` — push this repo as the umbrella. Set as default branch: `main`.
- [ ] `bare-swift/bare-swift-cli` — push the CLI repo.
- [ ] `bare-swift/swift-greet` — push the demo package.

For each: enable Discussions, enable Issues, enable Pages (source: GitHub Actions), enable branch protection on `main` (require PR, require status checks).

## Identity

- [ ] Replace placeholders in `MAINTAINERS.md` with the project lead's GitHub handle.
- [ ] Replace placeholder in `SECURITY.md` with a real security contact email.
- [ ] Generate a project GPG key for signing release tags. Add the public key to GitHub on the project lead's account. Document the key fingerprint in `MAINTAINERS.md`.

## Pages

- [ ] Enable GitHub Pages on `bare-swift/bare-swift` with source = "GitHub Actions".
- [ ] Verify after first push: https://bare-swift.github.io renders the generated `site/index.html`.
- [ ] For each package repo, enable Pages with source = "GitHub Actions" so DocC can publish to https://bare-swift.github.io/<package>/.

## SwiftPM Index

- [ ] After first package release, submit the org to https://swiftpackageindex.com so it auto-tracks all repos.

## Communications

- [ ] Create a `bare-swift` tag on the Swift Forums for cross-pollination.
- [ ] Set up GitHub Discussions categories: General, Q&A, Ideas, Show and tell, RFC discussion.

---

## Phase 0 status

- [x] Umbrella repo content authored
- [x] RFC-0001 + RFC-0002 landed
- [x] CLI tool (`bare-swift new` + `extract-vectors` + `gen-site`) built and tested
- [x] Shared CI / DocC / release workflows authored
- [x] Demo package `swift-greet` scaffolded, tested, consumed end-to-end

**Remaining for public Phase 0 launch (manual GH-side):** items above in this document.

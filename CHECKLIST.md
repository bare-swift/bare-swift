# bare-swift — manual setup checklist

This checklist tracks the state of the bare-swift GitHub presence. Items marked `[x]` are done; items marked `[ ]` still need human action (credentials, accounts, external services).

## Org

- [x] Create the GitHub organization `bare-swift`.
- [x] Add the project lead's account as the org owner.
- [ ] Set the org's public-facing email and homepage to point at this README / the future site.

## Repos created on GitHub

- [x] `bare-swift/bare-swift` — umbrella, default branch `main`.
- [x] `bare-swift/bare-swift-cli` — CLI tool, default branch `main`.
- [x] `bare-swift/swift-greet` — demo package, default branch `main`, tag `v0.1.0`.
- [x] `bare-swift/swift-hex` — first Phase 1 package, default branch `main`, tag `v0.1.0`.

For each:
- [x] Issues enabled.
- [x] Discussions enabled (umbrella only — packages keep issue-only).
- [x] Pages enabled (source: GitHub Actions).
- [x] **`github-pages` environment configured to allow `v*` tags + `main` branch** (required for tag-triggered DocC deploys). Run after `gh repo create`:
  ```bash
  REPO=swift-newpkg
  echo '{"deployment_branch_policy":{"protected_branches":false,"custom_branch_policies":true}}' \
    | gh api -X PUT "repos/bare-swift/$REPO/environments/github-pages" --input -
  echo '{"name":"main","type":"branch"}' | gh api -X POST "repos/bare-swift/$REPO/environments/github-pages/deployment-branch-policies" --input -
  echo '{"name":"v*","type":"tag"}'      | gh api -X POST "repos/bare-swift/$REPO/environments/github-pages/deployment-branch-policies" --input -
  ```
- [ ] Branch protection on `main` (require PR, require status checks). Run for each repo:
      `gh api -X PUT "repos/bare-swift/<repo>/branches/main/protection" --input <protection.json>`

## Identity

- [ ] Replace placeholders in `MAINTAINERS.md` with the project lead's GitHub handle.
- [ ] Replace placeholder in `SECURITY.md` with a real security contact email.
- [ ] Generate a project GPG key for signing release tags. Add the public key to GitHub on the project lead's account. Document the key fingerprint in `MAINTAINERS.md`.

## Pages — verified live

- [x] https://bare-swift.github.io/bare-swift/ — umbrella site (rendered from `packages/index.json`).
- [x] https://bare-swift.github.io/swift-greet/documentation/greet/ — DocC for swift-greet.
- [ ] (Future) https://bare-swift.github.io/bare-swift-cli/ — DocC for the CLI; not built in Phase 0 (CLI has no public library product to document).

## SwiftPM Index

- [ ] Submit the org to https://swiftpackageindex.com so it auto-tracks all repos. Form: https://swiftpackageindex.com/add-a-package

## Communications

- [ ] Create a `bare-swift` tag on the Swift Forums for cross-pollination.
- [ ] Set up GitHub Discussions categories on the umbrella repo: General, Q&A, Ideas, Show and tell, RFC discussion.

---

## Phase 0 status

- [x] Umbrella repo content authored
- [x] RFC-0001 + RFC-0002 landed
- [x] CLI tool (`bare-swift new` + `extract-vectors` + `gen-site`) built, tested, CI green
- [x] Shared CI / DocC / release workflows authored, fixed for macos-15, verified green
- [x] Demo package `swift-greet` scaffolded, tested, consumed end-to-end (local + via fresh SwiftPM project)
- [x] All three repos pushed to GitHub, public, with workflows running
- [x] swift-greet GitHub Release v0.1.0 created
- [x] Umbrella site live at https://bare-swift.github.io/bare-swift/
- [x] swift-greet DocC live at https://bare-swift.github.io/swift-greet/documentation/greet/
- [x] Scaffold templates updated to include `docs.yml` + `release.yml` + `swift-docc-plugin` so future packages get them by default

**Phase 0 is functionally complete.** Remaining items above are human-action: branch protection, GPG key, identity placeholders, SwiftPM Index submission, Forums tag.

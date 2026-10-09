# Changelog

## Unreleased

### Added

**`profiles/checks.yml`: the scanners, without the lint and test jobs.** A
repository whose own lint job does something the shared one cannot - depin
preflights a private module graph across six projects, names the ones whose
job-token allowlist is missing, and keeps no cache on purpose - had to choose
between that job and the whole template. It now takes `validate:*` alone and
keeps its own lint and test.


## v1.0.0 - 2026-09-18

First release. A `library` profile with `validate`, `lint` and `test` for Go
repositories that ship no container image.

The jobs are lifted from `aioz-stream`'s pipeline, the only complete
implementation the team had, rather than rewritten. Changes made while
extracting them:

- Tool invocations no longer go through `make`. No library repository has
  `make lint-verify` or `make test-build`, and `aioz-logger` has no Makefile at
  all; a template requiring them would onboard nobody. `GENERATE_VERIFY_CMD` and
  `TEST_BUILD_CMD` let a repository that has such targets wire them in.
- `-tags purego` became the `BUILD_TAGS` variable, defaulting to empty, with the
  `-tags` flag omitted entirely when empty.
- `.aioz-go-private` is opt-in. Only one of the four library repositories has a
  real private dependency.
- `validate:hadolint` exits 0 with a notice when the repository has no
  Dockerfile. `aioz-stream` errors in that case, which is right for a service
  and wrong for a library.
- `validate:trivy-fs` passes `--ignorefile` only when `.trivyignore.yaml`
  exists.
- `validate:haproxy` was left in `aioz-stream`. It is repo-specific.
- golangci-lint is pinned to `v2.12.2`, resolving a drift against
  team-playbook's copy-paste template, which pinned `v2.11.4` and installed via
  the upstream `install.sh` - whose non-anchored asset match downloads the SBOM
  instead of the tarball and dies on a checksum mismatch.
- Ref rules were narrowed to merge requests, the default branch and tags.
  `aioz-stream`'s `develop*` and `stag` are that repository's own concern.

The `validate` stage does not block. See "Why validate does not block" in
README.md.

# aioz-ci-template

Shared GitLab CI jobs for the team's Go repositories. A repository includes a
profile instead of copying YAML, so a fix to a check reaches every repository
through one tag bump.

The standard these jobs implement is written up in
[team-playbook `conventional/ci-cd.md`](https://gitlab.internal/tuan.quang.tran/team-playbook/-/blob/main/conventional/ci-cd.md).

## Quick start

```yaml
# .gitlab-ci.yml
include:
  - project: tuan.quang.tran/aioz-ci-template
    ref: v0.1.0
    file: profiles/library.yml
```

That is the whole file for a Go module that needs nothing else. It gives you:

| Stage | Job | Blocking? |
| --- | --- | --- |
| `lint` | `lint` - golangci-lint, code-quality report inline on the MR | yes |
| `validate` | `validate:govulncheck` | no, warns |
| `validate` | `validate:gitleaks` | no, warns |
| `validate` | `validate:trivy-fs` | no, warns |
| `validate` | `validate:hadolint` | no, warns |
| `test` | `test` - `go test -race ./...` | yes |

Jobs run on merge requests, the default branch, and tags.

**Blocking needs one project setting.** Tick **Settings -> Merge requests ->
"Pipelines must succeed"**. Without it a red pipeline is only a red icon next to
a live merge button.

## Profiles

| Profile | For |
| --- | --- |
| `profiles/library.yml` | Go modules that ship no container image: aioz-common, aioz-config, aioz-logger, aioz-stats, go-sdk. Also the right profile for the *checks* of a repository whose release flow is out of scope, such as a binary published over FTP. |

A `service` profile (image, migrate, deploy, rollback) comes once the library
profile is proven. It will build on the same hidden jobs.

Always name a profile, never a `jobs/` file. The `jobs/` layout is internal and
may be rearranged within a major version.

## Why validate does not block

Every `validate:*` job sets `allow_failure: true`, so a finding is a yellow
warning on the pipeline and in the merge request, and does not stop the merge.

This is deliberate and temporary. The older libraries carry a backlog of
findings nobody has triaged, and govulncheck in particular reports transitive
vulnerabilities that cannot be fixed from the repository reporting them - it has
no ignore file, so such a finding cannot even be suppressed. Blocking today
would leave those pipelines permanently red, which teaches people to ignore the
whole stage. That is the exact failure this template exists to remove.

A repository that has cleared its findings should enforce them, one job at a
time:

```yaml
validate:gitleaks:
  allow_failure: false
```

`lint` and `test` block today and have no such switch.

This is a per-job override rather than a `VALIDATE_ALLOW_FAILURE` variable
because GitLab requires `allow_failure` to be a boolean literal. A
variable-driven switch means duplicating every ref rule into a blocking and a
warning variant; one line per job reads honestly on the pipeline page.

## Variables

Override any of these in your own top-level `variables:` block, which takes
precedence over the included defaults.

| Variable | Default | Note |
| --- | --- | --- |
| `GO_IMAGE` | `registry:5000/go:1.25` | Must be go1.25.13+. govulncheck checks the standard library too, and go1.25.8 has ten reachable stdlib vulnerabilities. |
| `GOLANGCI_LINT_VERSION` | `v2.12.2` | Must be v2.x. A v2-schema `.golangci.yml` is rejected by a v1 binary. |
| `GOVULNCHECK_VERSION` | `v1.7.0` | Newest release that runs on Go 1.25. Move it with `GO_IMAGE`. |
| `GITLEAKS_IMAGE` | digest-pinned v8.30.1 | |
| `TRIVY_IMAGE` | digest-pinned 0.74.0 | |
| `HADOLINT_IMAGE` | digest-pinned v2.15.1 | |
| `BUILD_TAGS` | *(empty)* | Build tags the repository compiles under, e.g. `purego`. The `-tags` flag is omitted entirely when empty. |
| `RUNNER_TAG` | `linux` | |
| `GIT_SUBMODULE_STRATEGY` | `none` | Submodules are separate modules with their own pipelines. |
| `TRIVY_SKIP_DIRS` | `.cache` | Comma-separated. |
| `GO_PRIVATE_HOSTS` | `gitlab.internal/,10.0.0.50/` | Used only by `.aioz-go-private`. |
| `GENERATE_VERIFY_CMD` | *(empty)* | Run first in `test`. Should fail if generated output (swagger, proto) is stale. Unset means skipped. |
| `TEST_BUILD_CMD` | *(empty)* | Run second in `test`. Should compile test code behind build tags `go test` skips. Unset means skipped. |

No Makefile is required. The jobs call the tools directly, so a repository with
no Makefile at all can adopt the profile as-is. `GENERATE_VERIFY_CMD` and
`TEST_BUILD_CMD` are how a repository that *has* those targets wires them in.

## Private Go modules

Repositories importing a module from `gitlab.internal` or `10.0.0.50` must opt
into the credential setup, because most repositories need nothing:

```yaml
lint:
  before_script:
    - !reference [.aioz-go-private, before_script]
    - !reference [.aioz-lint, before_script]

test:
  before_script:
    - !reference [.aioz-go-private, before_script]
    - !reference [.aioz-test, before_script]
```

Order matters: the credentials must be in place before the job's own `go list`
fetch runs.

The project owning the private module must also allowlist the consuming project
under **Settings -> CI/CD -> Job token permissions -> Allowlist**, or the clone
returns 403 with a message that does not mention the allowlist.

## Adding repository-specific jobs

Keep them in your own `.gitlab-ci.yml` and extend the shared hidden jobs:

```yaml
stages: [lint, validate, test, integration]   # must list the shared three too

integration:
  extends: .aioz-test
  stage: integration
  script:
    - go test -race -timeout 20m ./redis/... ./postgres/...
```

**`stages:` replaces, it does not merge.** A repository adding a stage must
declare the shared three as well, or the shared jobs land in an undeclared
stage and the pipeline fails to create.

**`rules:` replaces too.** The shared jobs run on merge requests, the default
branch, and tags. A repository needing other refs - `aioz-stream` also uses
`develop*` and `stag` - redefines `rules:` on the job and must list every ref it
wants, including those three.

## Hidden jobs

| Name | Holds |
| --- | --- |
| `.aioz-rules` | The ref rules: merge requests, default branch, tags |
| `.aioz-go-private` | `before_script` for private-module credentials (opt-in) |
| `.aioz-lint` | Image, Go and lint caches, golangci-lint install and config verify |
| `.aioz-validate` | Stage, runner tag, and `allow_failure: true` |
| `.aioz-test` | Image, `CGO_ENABLED=1`, test cache, module fetch |

## No YAML anchors

Nothing in `jobs/` or `profiles/` uses a YAML anchor (`&name`) or alias
(`*name`), and the repository's own pipeline fails if one appears.

Anchors are resolved by the YAML parser before GitLab merges anything, so an
alias cannot see an anchor defined in another file. A template using one works
inside its own repository and breaks silently the moment a consumer includes it
- the key simply goes missing. `extends:` and `!reference` are resolved after
the merge and do work across files.

## Versioning

Tags are semver and consumers pin an exact tag. `main` is not a supported `ref:`
for a consumer: a change to it would reach every repository unreviewed.

| Change | Bump |
| --- | --- |
| A job renamed or removed, a variable's meaning changed, a check made blocking | major |
| A job or profile added, a tool version bumped, a new variable with a default | minor |
| A comment, a doc fix, a bug fix that changes no interface | patch |

`v0.x` means the interface may still move on a minor bump while the library
profile is being proven. `v1.0.0` follows the first successful pilot.

## Developing

```sh
yamllint -c .yamllint.yml .gitlab-ci.yml jobs/ profiles/
grep -rnE '(^|[[:space:]])[&*][A-Za-z_]' jobs/ profiles/   # must find nothing
glab ci lint                                               # if glab is installed
```

The repository's own pipeline runs all three, plus GitLab's CI Lint endpoint
against every profile expanded exactly as a consumer would get it. That last
check needs a PAT with `api` scope in the masked CI/CD variable
`CI_LINT_TOKEN`; without it the job explains itself and passes.

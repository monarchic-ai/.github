# Monarchic Meta GitHub Defaults

Shared GitHub Actions workflows and organization defaults for Monarchic
repositories.

## Reusable Workflows

### Nix CI

Use `.github/workflows/nix-ci.yml` from repositories with a Nix flake:

```yaml
name: Nix CI

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  nix-ci:
    uses: monarchic-ai/.github/.github/workflows/nix-ci.yml@main
    with:
      publish_cache: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
    secrets: inherit
```

The reusable workflow runs on the shared self-hosted NixOS runner labels,
builds every `packages.${system}` output, builds every `checks.${system}`
output, runs `nix flake check`, and optionally signs and uploads built store
paths to the Monarchic Nix binary cache. It is intended to run for trusted
pushes to `main`, including merges, and for manual dispatches. If the caller
repository has a `MONARCHIC_GITHUB_PAT` secret, the workflow uses it for private
flake inputs.

### Release

Use `.github/workflows/release.yml` as a preflight job from repository release
workflows before publishing packages, images, deployments, or GitHub releases:

```yaml
jobs:
  release-preflight:
    uses: monarchic-ai/.github/.github/workflows/release.yml@main
    secrets: inherit
```

The reusable workflow runs on the shared self-hosted NixOS runner labels,
requires a tag ref, checks that the tag matches `v*.*.*` by default, verifies
that the tag points at `origin/main`, and verifies that the caller repository
already has a successful `Nix CI` workflow run for the tagged commit. It
performs no publishing or deployment itself; caller workflows keep repo-specific
release steps behind this preflight job.

### NPM Publish

Use `.github/workflows/npm-publish.yml` from Node package repositories whose
package build and tests are already covered by `nix-ci.yml`:

```yaml
name: Publish npm package

on:
  push:
    tags: [v*.*.*]

jobs:
  publish:
    uses: monarchic-ai/.github/.github/workflows/npm-publish.yml@main
    secrets: inherit
```

The reusable workflow runs the shared release preflight first, then installs
dependencies, builds the package, and publishes to npm with `NPM_TOKEN`. It
does not rerun repository tests; release tags should point at commits that
already passed `Nix CI`.

### Maintenance

Use `.github/workflows/maintenance.yml` from scheduled or manual caller
workflows that should produce a report before optional generated-file updates:

```yaml
name: Maintenance

on:
  workflow_dispatch:
  schedule:
    - cron: "17 9 * * 1"

jobs:
  maintenance:
    uses: monarchic-ai/.github/.github/workflows/maintenance.yml@main
    with:
      check_flake: true
      maintenance_command: nix run .#maintenance-report
    secrets: inherit
```

The reusable workflow runs on the shared self-hosted NixOS runner labels,
optionally evaluates `nix flake check --no-build`, optionally runs a repo-local
maintenance command, writes a Markdown report to the job summary, and uploads it
as an artifact. It does not push commits, open pull requests, deploy, publish,
or mutate external systems by default. Callers that need generated-file
maintenance can pass `commit_paths`, `commit_message`, and `push_branch`; only
those configured paths are staged and pushed. To open or update a generated
maintenance pull request instead of pushing directly to the current ref, also
pass `create_pull_request`, `checkout_ref`, `pull_request_base_branch`,
`pull_request_title`, `pull_request_body`, and optional `pull_request_labels`.

### Webapp Release Smoke

Use `.github/workflows/webapp-release-smoke.yml` from webapp repositories that
expose release-smoke commands through Nix apps:

```yaml
name: Webapp release smoke

on:
  workflow_dispatch:

jobs:
  release-smoke:
    uses: monarchic-ai/.github/.github/workflows/webapp-release-smoke.yml@main
    secrets: inherit
```

The reusable workflow is manual-only from caller repositories. It runs on the
shared self-hosted NixOS runner labels, installs dependencies with the caller
flake dev shell, runs staging infra readiness through caller-provided Nix apps
when strict non-dry staging gates are enabled, and runs `nix run
.#webapp-release-smoke` for the aggregate release smoke. Caller workflows keep
only dispatch inputs and delegate the implementation here.

### Docker CI

Use `.github/workflows/docker-ci.yml` only for repositories where Docker is
still an explicit build or runtime contract:

```yaml
name: Docker CI

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  docker-ci:
    uses: monarchic-ai/.github/.github/workflows/docker-ci.yml@main
    with:
      runner_labels: '["self-hosted","Linux","X64","monarchic-local","docker"]'
      context: .
      dockerfile: Dockerfile
      image_name: example-docker-ci
      smoke_command: docker run --rm example-docker-ci --version
```

The reusable workflow requires a Docker-capable runner label set, verifies
Docker availability, builds the requested image, and optionally runs a smoke
command. Prefer moving build, test, lint, smoke, and security behavior into
flake checks and using `nix-ci.yml` when Docker is not part of the repo's real
contract.

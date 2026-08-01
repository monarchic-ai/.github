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

The reusable workflow runs on `vars.CI_RUNNER_LABELS`, builds every
`packages.${system}` output, builds every `checks.${system}` output, runs
`nix flake check`, and optionally signs and uploads built store paths to the
Monarchic Nix binary cache. It is intended to run for trusted pushes to `main`,
including merges, and for manual dispatches. If the caller repository has a
`MONARCHIC_GITHUB_PAT` secret, the workflow uses it for private flake inputs.

### Release

Use `.github/workflows/release.yml` as a preflight job from repository release
workflows before publishing packages, images, deployments, or GitHub releases:

```yaml
jobs:
  release-preflight:
    uses: monarchic-ai/.github/.github/workflows/release.yml@main
    secrets: inherit
```

The reusable workflow runs on `vars.CI_RUNNER_LABELS`, requires a tag ref,
checks that the tag matches `v*.*.*` by default, verifies that the tag points at
`origin/main`, and verifies that the caller repository already has a successful
`Nix CI` workflow run for the tagged commit. It performs no publishing or
deployment itself; caller workflows keep repo-specific release steps behind
this preflight job.

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
workflows that should produce a report before any automation becomes mutating:

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

The reusable workflow runs on `vars.CI_RUNNER_LABELS`, optionally evaluates
`nix flake check --no-build`, optionally runs a repo-local maintenance command,
writes a Markdown report to the job summary, and uploads it as an artifact. It
does not push commits, open pull requests, deploy, publish, or mutate external
systems by default.

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

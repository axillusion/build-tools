# build-tools

A collection of reusable GitHub Actions workflows for CI/CD automation. These workflows are designed to be generic, composable, and organization-independent.

## Overview

This repository provides reusable workflows that can be called from other repositories using GitHub Actions `workflow_call`. They cover common tasks like Docker builds, Helm chart releases, and Go testing.

## Workflows

### Docker Build and Push (`docker-build-and-push.yml`)

Builds and pushes Docker images to a registry with multi-architecture support.

**Purpose:** Automate Docker image building and publishing for multiple platforms (e.g., amd64, arm64).

**Usage:**

```yaml
jobs:
  docker:
    uses: axillusion/build-tools/.github/workflows/docker-build-and-push.yml@main
    permissions:
      contents: read
      packages: write
    with:
      image: ghcr.io/org/app
      context: "."
      dockerfile: "Dockerfile"
      platforms: "linux/amd64,linux/arm64"
      add_latest: true
    secrets:
      registry_token: ${{ secrets.GITHUB_TOKEN }}
```

**Inputs:**

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `image` | Full image name (e.g., `ghcr.io/org/app`) | ✅ Yes | — |
| `context` | Build context | No | `"."` |
| `dockerfile` | Path to Dockerfile | No | `"Dockerfile"` |
| `platforms` | Target platforms | No | `"linux/amd64,linux/arm64"` |
| `version` | Version tag (omit to derive from Git tag) | No | — |
| `add_latest` | Also push `:latest` tag | No | `true` |
| `build_secret` | Optional BuildKit secret (e.g., `id=m2,src=./settings.xml`) | No | — |
| `registry_token_secret` | Name of the secret containing registry token | No | `"GITHUB_TOKEN"` |

**Secrets:**

| Secret | Description | Required |
|--------|-------------|----------|
| `registry_token` | Registry authentication token | ✅ Yes |

Pass your repository's secret (e.g., `GITHUB_TOKEN`, `GIT_TOKEN`, or custom token):

```yaml
secrets:
  registry_token: ${{ secrets.GIT_TOKEN }}
```

**Permissions Required:**
- `contents: read` — to checkout the repository
- `packages: write` — to push images to GHCR

**Example with custom token secret:**

```yaml
jobs:
  docker:
    uses: axillusion/build-tools/.github/workflows/docker-build-and-push.yml@main
    permissions:
      contents: read
      packages: write
    with:
      image: ghcr.io/myorg/app
      version: "1.2.3"
    secrets:
      registry_token: ${{ secrets.GIT_TOKEN }}
```

---

### Publish Helm Chart (`publish-helm-chart.yml`)

Packages and publishes Helm charts to an OCI registry (e.g., GHCR).

**Purpose:** Automate Helm chart releases with optional linting and dependency management.

**Usage:**

```yaml
jobs:
  helm:
    uses: axillusion/build-tools/.github/workflows/publish-helm-chart.yml@main
    permissions:
      contents: read
      packages: write
    with:
      chart_dir: "./charts/myapp"
      registry: "ghcr.io"
      repository: "myorg/charts"
```

**Inputs:**

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `chart_dir` | Path to Helm chart folder (must contain Chart.yaml) | ✅ Yes | — |
| `registry` | OCI registry URL | No | `"ghcr.io"` |
| `repository` | OCI repository path (no tag) | No | `"doda2025-team20/team20-helm"` |
| `helm_version` | Helm version to use | No | `"v3.15.4"` |
| `lint` | Run `helm lint` | No | `true` |

**Permissions Required:**
- `contents: read` — to checkout the repository
- `packages: write` — to push charts to OCI registry

**Example with custom registry:**

```yaml
jobs:
  helm:
    uses: axillusion/build-tools/.github/workflows/publish-helm-chart.yml@main
    permissions:
      contents: read
      packages: write
    with:
      chart_dir: "./helm"
      registry: "registry.example.com"
      repository: "myorg/charts"
      helm_version: "v3.14.0"
      lint: true
```

---

### Go Test (`go-test.yml`)

Runs Go tests with optional race detection and coverage checking.

**Purpose:** Validate Go code by running tests and checking code coverage.

**Usage:**

```yaml
jobs:
  test:
    uses: axillusion/build-tools/.github/workflows/go-test.yml@main
    with:
      go_version: "1.23"
      test_flags: "-race"
```

**Inputs:**

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `go_version` | Go version to use | No | `"1.23"` |
| `working_directory` | Working directory for tests | No | `"."` |
| `test_flags` | Flags to pass to `go test` (e.g., `-race`, `-shuffle=on`) | No | `"-race"` |
| `coverage_threshold` | Minimum coverage percentage required (0 to disable) | No | `"0"` |

**Permissions Required:**
- `contents: read` — to checkout the repository

**Example with coverage threshold:**

```yaml
jobs:
  test:
    uses: axillusion/build-tools/.github/workflows/go-test.yml@main
    with:
      go_version: "1.22"
      test_flags: "-race -shuffle=on"
      coverage_threshold: "80"
```

**Example with monorepo subdirectory:**

```yaml
jobs:
  test:
    uses: axillusion/build-tools/.github/workflows/go-test.yml@main
    with:
      go_version: "1.23"
      working_directory: "./services/api"
      test_flags: "-race"
```

---

### Publish Markdown to XWiki (`publish-xwiki.yml`)

Publishes the caller's Markdown using a prebuilt `md2xwiki` image. The
publisher compiles the Markdown, checks the deployment plan, uploads changes,
and verifies the result. Use a versioned image or digest.

```yaml
jobs:
  publish:
    uses: axillusion/build-tools/.github/workflows/publish-xwiki.yml@main
    permissions:
      contents: read
      packages: read
    with:
      image: ghcr.io/example/md2xwiki:0.1.0
      config: xwiki.toml
      environment: xwiki
      artifact_name: xwiki-${{ github.run_id }}-${{ github.run_attempt }}
    secrets:
      XWIKI_USERNAME: ${{ secrets.XWIKI_USERNAME }}
      XWIKI_PASSWORD: ${{ secrets.XWIKI_PASSWORD }}
```

| Input | Description | Default |
| --- | --- | --- |
| `image` | Prebuilt `md2xwiki` image with a tag or digest | Required |
| `config` | Repository-relative TOML file selecting source pages and wiki destinations | `xwiki.toml` |
| `output` | Repository-relative output directory | `build/xwiki` |
| `environment` | Deployment environment in the caller repository | Empty |
| `login_to_ghcr` | Authenticate to GHCR with the caller's job token | `false` |
| `use_vpn` | Connect to OpenVPN after pulling the image | `false` |
| `overwrite` | Back up and replace drifted managed content | `false` |
| `prune` | Delete unchanged, previously owned resources omitted from source | `false` |
| `artifact_name` | Upload compilation, backups, and summaries under this artifact name | Empty |

Pass `XWIKI_USERNAME` and `XWIKI_PASSWORD` as secrets. If the deployment
environment holds them, use the same secret names and pass them explicitly as
shown. Set `XWIKI_EXPECTED_USER` as a repository or environment variable when
the authenticated XWiki reference differs from the username.

To use OpenVPN, add `use_vpn: true` under `with` and
`OPENVPN_CONFIG: ${{ secrets.OPENVPN_CONFIG }}` under `secrets`. Keep the
complete non-interactive profile, including any inline keys and certificates,
in that secret. The job fails if the profile is blank, and it disconnects and
removes the profile after publishing. A run with `use_vpn: false` requires no
profile.

The workflow uses `contents: read` to check out the caller's files and
`packages: read` for optional GHCR authentication. For a private package,
enable `login_to_ghcr` and give the caller repository read access to that
package. The workflow exposes no job outputs. If requested, its artifact
contains the compiled content, deployment summaries, and any wiki backups.
Choose a distinct artifact name for each call in the same workflow run, and
restrict access to runs whose artifacts contain private wiki content.

---

## Complete Example Workflow

Here's a complete example that uses all three workflows in a single caller workflow:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
    tags:
      - "v*"

jobs:
  test:
    uses: axillusion/build-tools/.github/workflows/go-test.yml@main
    with:
      go_version: "1.23"
      test_flags: "-race"
      coverage_threshold: "75"

  docker:
    needs: test
    uses: axillusion/build-tools/.github/workflows/docker-build-and-push.yml@main
    permissions:
      contents: read
      packages: write
    with:
      image: ghcr.io/myorg/myapp
      platforms: "linux/amd64,linux/arm64"
      add_latest: true
    secrets:
      registry_token: ${{ secrets.GITHUB_TOKEN }}

  helm:
    needs: docker
    uses: axillusion/build-tools/.github/workflows/publish-helm-chart.yml@main
    permissions:
      contents: read
      packages: write
    with:
      chart_dir: "./charts/myapp"
      registry: "ghcr.io"
      repository: "myorg/charts"
```

---

## Repository Principles

This repository follows these principles:

- **Generic** — No hardcoded organization names or project-specific assumptions
- **Reusable** — Designed to be called from any repository
- **Composable** — Workflows can be combined and used independently
- **Organization-independent** — Works for personal projects, organizations, and open-source
- **Language-agnostic** — Generic workflows avoid language-specific assumptions

---

## Security

All workflows follow security best practices:

- ✅ No hardcoded secrets
- ✅ Secrets passed via caller repository or organization secrets
- ✅ Minimal required permissions
- ✅ Credentials never logged or exposed

---

## Versioning

Pin workflows to a specific version for stability:

```yaml
# Pin to a specific tag
uses: axillusion/build-tools/.github/workflows/go-test.yml@v1.0.0

# Pin to a branch
uses: axillusion/build-tools/.github/workflows/go-test.yml@main

# Pin to a commit SHA
uses: axillusion/build-tools/.github/workflows/go-test.yml@abc123def456
```

---

## Contributing

When extending this repository:

- Keep workflows generic and reusable
- Avoid organization-specific behavior
- Document all inputs and required permissions
- Preserve backward compatibility where possible

See [AGENTS.md](./AGENTS.md) for detailed design guidelines.

---

## License

These workflows are provided as-is for use in your GitHub repositories.

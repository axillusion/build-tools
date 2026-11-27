# build-tools

A collection of reusable GitHub Actions workflows for CI/CD automation. These workflows are designed to be generic, composable, and organization-independent.

## Overview

This repository provides reusable workflows that can be called from other repositories using GitHub Actions `workflow_call`. They cover semantic releases, Docker builds, Helm chart releases, and Go testing.

## Workflows

### Semantic Release (`semantic-release.yml`)

Checks the pull request title and every non-merge commit introduced by the pull
request. On pushes to the default branch it checks every non-merge commit since
the highest reachable `vMAJOR.MINOR.PATCH` tag, then uses `semantic-release` to
calculate the next version and generate release notes.

| Change | Version bump |
| --- | --- |
| `!` in the commit header or a `BREAKING CHANGE:` or `BREAKING-CHANGE:` footer | Major |
| `feat:` | Minor |
| `fix:`, `perf:`, or a recognized revert of a previously released change | Patch |
| Other types, such as `docs:`, `test:`, and `chore:` | No release on their own |

The highest applicable bump wins. The first release is `1.0.0` if there is no
prior version tag. To adopt the workflow in a repository with legacy commit
messages, tag the last released commit with its `vMAJOR.MINOR.PATCH` version.
For an untagged repository, set `initial_validation_base` to the last legacy
commit's full SHA to start validation after that commit. Otherwise, validation
covers the full default-branch history before the first version tag.

The semantic release calculates one version for the repository. By default it
publishes a GitHub Release with a generated changelog grouped by commit type,
including features, bug fixes, documentation, and breaking changes.

| Optional artifact | How to enable it | What it publishes |
| --- | --- | --- |
| Docker image | Set `image` to the full GHCR destination | The image tagged `MAJOR.MINOR.PATCH` and, by default, `latest` |
| Helm chart | Set `helm_chart_dir` and `helm_repository` | An OCI chart packaged at `MAJOR.MINOR.PATCH` |

Both artifacts use the version calculated by the semantic release. To publish
artifacts with a Git tag alone, set `github_release: false` and configure at
least one artifact. The workflow creates the `vMAJOR.MINOR.PATCH` Git tag
after the configured registry jobs succeed. That tag is the baseline for the
next release. A run with no version bump skips publishing. Use **Re-run all
jobs** to retry from the current tags after a failure.

```yaml
name: Release

on:
  pull_request:
    types: [opened, synchronize, reopened, edited, ready_for_review]
  push:
    branches: [main]

jobs:
  release:
    uses: axillusion/build-tools/.github/workflows/semantic-release.yml@main
    permissions:
      contents: write
      packages: write
    with:
      image: ghcr.io/myorg/myapp
      helm_chart_dir: charts/myapp
      helm_repository: myorg/charts
```

This example adds both artifacts to the semantic release. `helm_repository`
is the namespace path in the registry. Helm appends the name from
`Chart.yaml`, so a chart named `myapp` goes to `oci://ghcr.io/myorg/charts/myapp` with the
calculated chart version. [Helm OCI publishing](https://helm.sh/docs/topics/registries/#the-push-subcommand)

Set the push filter to your default branch. Commit headers use the type list
below, an optional named scope, an optional `!`, and a required description.
For example, `feat(api)!: remove the legacy endpoint`.
Separate a commit body or footer from the header with a blank line. If you use
squash merging, configure GitHub to use the pull request title as the commit
title and keep the conventional header when merging.

The caller grants `contents: write` for Git tags and GitHub releases and
`packages: write` for GHCR. The pull request check runs with `contents: read`.
The workflow defaults to the caller's built-in `GITHUB_TOKEN` for GitHub and
GHCR. For an existing GHCR package, grant the caller repository write access
or pass the corresponding `docker_registry_token` or `helm_registry_token`
secret. For a Helm registry outside GHCR, set `helm_registry` and
`helm_registry_username` and pass `helm_registry_token`.

| Input | Description | Default |
| --- | --- | --- |
| `github_release` | Publish a GitHub Release with the grouped changelog | `true` |
| `initial_validation_base` | Full ancestor commit SHA. Validate later commits on the release branch until the first version tag exists | Empty |
| `image` | Full GHCR image name without a tag or digest | Empty to skip Docker |
| `context` | Docker build context | `.` |
| `dockerfile` | Path to the Dockerfile | `Dockerfile` |
| `platforms` | Target container platforms | `linux/amd64,linux/arm64` |
| `add_latest` | Also publish the `latest` container tag | `true` |
| `build_secret` | BuildKit secret argument forwarded to the Docker workflow | Empty |
| `helm_chart_dir` | Helm chart directory containing `Chart.yaml` | Empty to skip Helm |
| `helm_registry` | Helm OCI registry hostname | `ghcr.io` |
| `helm_repository` | Helm OCI namespace path, for example `myorg/charts` | Required with `helm_chart_dir` |
| `helm_registry_username` | Helm OCI login username | GitHub actor |

The outputs `version` (for example `1.2.3`) and `tag` (for example `v1.2.3`)
are set after the configured targets finish and the Git tag is created.

#### Enforce commit messages with a GitHub ruleset

For organizations on a GitHub Enterprise plan, a branch ruleset can reject
updates with commit messages that fail a pattern. In the repository's
**Settings** sidebar, open **Rulesets**, select **New branch ruleset**, and
target the default branch. Under **Restrictions**, add **Restrict commit
metadata**. Select **Commit message** and **Must match a given regex pattern**,
then paste the RE2 pattern below. Set the ruleset to **Active** and save it.
[GitHub ruleset setup](https://docs.github.com/en/enterprise-cloud@latest/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/creating-rulesets-for-a-repository#adding-metadata-restrictions)

```regex
\A(?:build|chore|ci|docs|feat|fix|perf|refactor|revert|style|test)(?:\([^()\r\n]+\))?!?: \S[^\r\n]*(?:\r?\n[ \t]*\r?\n[\s\S]*)?(?:\r?\n)?\z
```

The ruleset and the workflow use Commitlint's conventional type list.
For example, `feat(api): add search`, `chore: refresh tooling`, and
`fix: handle a timeout` pass. `update: refresh tooling` fails because `update`
is outside that list. Use `feat:` for an unscoped feature and `feat(api):`
when a scope adds context. Both forms require a description.
[Commitlint type list](https://github.com/conventional-changelog/commitlint/blob/master/%40commitlint/config-conventional/src/index.ts#L20-L35)

GitHub's generated `Merge pull request ...` subject fails
this rule. Use a conventional subject for merge commits or use squash or rebase
merging.

To check PR titles or to enforce the policy when **Restrict commit metadata**
is unavailable, require pull requests and the `release / Conventional Commits`
status check on the target branch.
[GitHub ruleset availability](https://docs.github.com/en/enterprise-cloud@latest/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets#metadata-restrictions)

---

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

`build-tools` runs its own semantic release on `main` to publish GitHub releases
and version tags. Each tag identifies a snapshot of all reusable workflows.
Pin a workflow to a published tag to keep using that version as `main` evolves.

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

# AGENTS.md

## Purpose

This repository contains reusable GitHub Actions workflows and CI/CD utilities shared across multiple repositories.

The primary goals are:

* reusable Docker build and publish workflows
* reusable Helm chart release workflows
* reusable CI/CD helper workflows
* organization-independent automation tooling

The workflows in this repository are intended to be consumed through GitHub Actions `workflow_call`.

Example:

```yaml
jobs:
docker:
uses: axillusion/build-tools/.github/workflows/docker-build-and-push.yml@main
```

---

# Repository Philosophy

This repository should remain:

* generic
* reusable
* composable
* organization-independent
* language-agnostic where possible

Avoid introducing:

* hardcoded organization names
* project-specific assumptions
* embedded infrastructure dependencies
* mandatory language-specific setup

Bad example:

```yaml
https://maven.pkg.github.com/doda2025-team20/lib-version
```

Good example:

* configurable inputs
* optional secrets
* optional setup steps
* caller-controlled behavior

---

# Workflow Design Rules

## Generic workflows stay generic

Generic reusable workflows must not assume:

* Java
* Maven
* NodeJS
* Python
* internal registries
* organization-specific infrastructure

Language-specific behavior should either:

* be optional
* or exist in dedicated workflows

Preferred structure:

```text
.github/workflows/
├── docker-build-and-push.yml
├── helm-release.yml
├── java-docker-build.yml
├── node-docker-build.yml
```

---

# Docker Workflow Guidelines

Docker workflows should support:

* multi-architecture builds
* configurable Dockerfiles
* configurable build contexts
* configurable tags
* configurable image registries
* optional BuildKit secrets

Preferred features:

* semantic version support
* optional latest tags
* optional build secrets
* reusable image naming

Avoid:

* mandatory Maven configuration
* embedded credentials
* hardcoded package registries
* organization-specific URLs

---

# Helm Workflow Guidelines

Helm workflows should:

* support OCI registries
* support configurable chart paths
* support configurable chart versions
* avoid chart-specific assumptions
* avoid organization-specific registries

---

# Security Requirements

## Never hardcode secrets

Do not commit:

* passwords
* tokens
* PATs
* registry credentials
* internal secrets

Secrets must come from:

* caller repository secrets
* organization secrets
* inherited reusable workflow secrets

---

## Least privilege principle

Workflows should request only required permissions.

Example:

```yaml
permissions:
contents: read
packages: write
```

Avoid broad permissions unless explicitly required.

---

# Reusable Workflow Compatibility

Reusable workflows are infrastructure interfaces.

Changes may affect many repositories.

When modifying workflows:

* preserve backward compatibility where possible
* prefer additive changes
* avoid silently changing behavior
* avoid breaking existing inputs

If introducing breaking changes:

* create a new workflow
* or require explicit migration

---

# Versioning Expectations

Consumers may pin workflows by:

* branch
* tag
* commit SHA

Agents should avoid introducing unexpected behavioral changes to existing workflows.

---

# Documentation Requirements

Each reusable workflow should document:

* purpose
* required permissions
* required secrets
* inputs
* outputs
* example usage

Example:

```yaml
jobs:
release:
uses: axillusion/build-tools/.github/workflows/docker-build-and-push.yml@main
permissions:
contents: read
packages: write
with:
image: ghcr.io/example/app
```

---

# Repository Structure

Expected structure:

```text
.github/
└── workflows/
├── docker-build-and-push.yml
├── helm-release.yml
└── ...
```

---

# Shell Script Guidelines

Shell scripts inside workflows should:

* fail fast
* avoid hidden assumptions
* prefer POSIX-compatible syntax where possible
* keep logic readable and explicit

Prefer:

* clear variable naming
* defensive checks
* explicit error messages

Avoid:

* silent failures
* unnecessary complexity
* tightly coupled logic

---

# Agent Guidance

When extending this repository:

* prefer composability
* prefer configurability
* minimize assumptions
* keep workflows independent
* avoid unnecessary coupling
* avoid organization-specific behavior

This repository should remain usable for:

* personal projects
* organizations
* open-source projects
* private repositories
* multi-language environments

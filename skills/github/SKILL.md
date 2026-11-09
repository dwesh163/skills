---
name: github
description: CI/CD pipeline conventions for any language — lint and build defined once as reusable workflows, a container image built and pushed per pull request, and a release triggered by a version bump on the main branch (skip if already released), with a hardened GitHub Actions baseline (deny-by-default permissions, pinned actions, no untrusted input in run scripts, Dependabot, zizmor) and a GitLab CI port. Use when writing, reviewing or securing GitHub Actions workflows or .gitlab-ci.yml, setting up automated releases or image publishing, or adding a deployment step after a release.
metadata:
  tags: ci-cd, github-actions, gitlab-ci, docker, release, security, dependabot, supply-chain
---

# CI/CD

The pipeline shape is the same for every language; only the lint, test and build commands change. Use the project's own toolchain for those steps and keep everything else identical across repositories.

## Pipeline shape

**On a pull request to the main branch:**

1. **prepare**: compute an image tag `<sanitized-branch>-<short-sha>`.
2. **lint**: formatter check, linter, type check, tests. Fails the pipeline. A non-failing machine-readable lint report may be uploaded as an artifact.
3. **build**: needs `lint`. Builds and pushes the container image with the PR tag. Skipped for PRs from forks (they have no secrets).

**On push to the main branch:**

1. **lint**: same reusable definition as for PRs.
2. **detect-version**: read the version from the project's manifest or version file, validate it against a semver pattern, and check whether a release for it already exists.
3. **build**: only if that version is not released yet. Pushes `latest` and the version tag.
4. **create-release**: only if the build reports success. Creates the tag and the release with notes generated from the tagged commits since the previous release (see the `git` skill for the tags the generator buckets).

Consequences:

- Merging PRs never releases by itself. Pushing a `[version]` commit that bumps the manifest is the release. No manual release step.
- A release that is half pushed must finish: release runs never cancel each other. PR runs cancel their own superseded runs.

**Extensions** (deploying, bumping a downstream chart repository, notifying) are new jobs appended after `create-release` with `needs` on it. Never change the reusable lint/build contract or the version-detection outputs to feed one more job.

## Shared composite actions

Version detection and release-note generation come from shared composite actions reused by every repository (`dwesh163/actions`: `detect-version`, `release`, `bump-chart`), not hand-rolled shell steps per project. `detect-version` takes a project type matching the manifest (a JS manifest, a Python setup file, or a plain version file for any other language). Pin those actions like any third-party action.

## Registry

- Default: the forge's own container registry, image path `<registry>/<owner>/<repo>` lowercased, authenticated with the job's own token.
- An external registry uses a robot account scoped to the one repository, push-only, with its credentials passed by name.

## Platform details

- GitHub Actions: workflow layout, the security baseline, and keeping it hardened → [references/github-actions.md](references/github-actions.md)
- GitLab CI: porting the same shape → [references/gitlab-ci.md](references/gitlab-ci.md)

# GitHub Actions

## Contents

- Workflow files
- Security baseline
- Cross-repository jobs
- Keeping it hardened
- Repository settings

## Workflow files

Always four files, even on a small project:

| File | Role |
|---|---|
| `_lint.yaml` | Reusable (`workflow_call`). Checkout, toolchain setup, dependency install from the lockfile, lint / type check / tests, optional report artifact |
| `_build.yaml` | Reusable. Inputs `tags` and `version`; output `build_success`. Sets the version in the manifest, sets up Buildx, logs in, builds and pushes with the GitHub Actions cache, then marks success |
| `pr.yaml` | `pull_request` on the main branch. `prepare` → `lint` → `build` |
| `release.yaml` | `push` on the main branch. `lint` + `detect-version` → `build` → `create-release` |

Collapsing them into one file loses the reuse (both entry workflows call the same lint and build) for no gain. Start split rather than migrate later.

## Security baseline

Apply all of these to every workflow file, and keep applying them when editing one.

1. **Deny by default.** `permissions: {}` at the top of every file. Each job requests exactly what it needs: `contents: read` to check out, `packages: write` only on the job that pushes the image, `contents: write` only on the job that creates the release. A job calling a reusable workflow sets the ceiling for it, so the caller job must grant what the callee needs.
2. **Pin every external action** to a version tag, kept current by Dependabot. Look up the current tag (`gh api repos/<owner>/<repo>/releases/latest --jq .tag_name`), do not recall it. Never a branch ref on someone else's action. Tags can be moved by a compromised maintainer, so jobs holding publishing secrets may pin a full commit SHA with the tag as a trailing comment. Own composite actions without tags: tag them and pin the tag as soon as possible. Local reusable workflow calls need no pin.
3. **Untrusted text never goes inside `run:`.** An expression in a script is pasted in before the shell parses it, so a crafted branch name becomes code. Branch names, PR titles and bodies, commit messages, and anything derived from them (step outputs, reusable-workflow inputs) are passed through `env:` and read as quoted shell variables. Expressions are fine in `with:`, `if:` and `env:` values.
4. **`persist-credentials: false`** on every checkout that does not push, so the token is not left in the git config for later steps or uploaded artifacts.
5. **`pull_request`, never `pull_request_target`,** for anything that builds PR code. Fork PRs run without secrets and with a read-only token, so gate the push job on the PR head repository being this repository: forks are still linted, just not pushed.
6. **Secrets by name, never `secrets: inherit`.** A reusable workflow declares the secrets it needs under `workflow_call.secrets`; the caller passes exactly those.
7. **Bound every job.** `timeout-minutes` on every job (the default is six hours). `concurrency`: PR runs grouped per ref with cancellation, release runs in one group without cancellation.
8. **Validate what becomes a tag or a command.** The detected version is matched against a semver pattern before reaching an image tag, a release name or a CLI call. Sanitize branch names into tag-safe characters.
9. **Pin runner images** to a versioned label rather than `-latest`.

## Cross-repository jobs

A job that writes to another repository (bumping a chart, opening a PR elsewhere) authenticates with a GitHub App token, not the injected token, which is scoped to the current repository. Install the App on the target repository only, with the minimum permission. The job itself declares `permissions: {}`. Use an App token only where a job needs cross-repository access, never by default.

## Keeping it hardened

- **Dependabot** with the `github-actions` ecosystem (bumps tags and SHA pins with their comment) and the `docker` ecosystem (base images), weekly, with a `cooldown` of about a week so a freshly compromised release is not adopted the day it appears. Add the project's own package ecosystem as well.
- **zizmor** audits workflows statically (unpinned uses, template injection, excessive permissions, persisted credentials, inherited secrets, dangerous triggers). Its default policy demands SHA pins everywhere: when staying on tags, configure the `unpinned-uses` rule with a `ref-pin` policy in `.github/zizmor.yml`. Run it locally and in CI.
- **CODEOWNERS** on `/.github/` so any workflow change needs a maintainer's review.

## Repository settings

Not files, so easy to forget:

- Default `GITHUB_TOKEN` permissions: read repository contents.
- Allowed actions: GitHub-owned and verified creators, or an explicit allowlist.
- Approval required for workflows from outside contributors.
- A ruleset on the main branch: PR required, the lint check required, no force push.
- A ruleset restricting who may create release tags.

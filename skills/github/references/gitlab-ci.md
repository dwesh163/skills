# GitLab CI

Port the pipeline *shape*, not the GitHub Actions syntax: one `.gitlab-ci.yml` with four stages, `lint` → `detect` → `build` → `release`.

- **lint**: the project's lint, type check and tests, in an image of the project's runtime.
- **detect**: read and validate the version, ask the GitLab Releases API whether that release exists, and pass `VERSION` and a `SHOULD_BUILD` flag to later jobs through a `dotenv` report artifact.
- **build**: runs only when `SHOULD_BUILD` is true. Docker-in-Docker as a service (reached over its service hostname, not a mounted socket), registry login with the password on stdin, push `latest` and the version tag.
- **release**: creates the release through the same Releases API, with the same tagged-commit notes.

Rules that carry over from the GitHub baseline:

- Pin every image, including the Docker and Docker-in-Docker images, to a version.
- Credentials in masked, protected CI/CD variables, scoped to the protected main branch; never echoed.
- Untrusted values (branch names, commit messages) quoted as shell variables, never spliced into commands.
- `rules:` restrict the build and release jobs to the main branch and merge requests from the project itself.
- Every job gets a `timeout`; release jobs are `interruptible: false`, merge-request jobs `interruptible: true`.

When a runner image cannot run the project's preferred toolchain (missing CPU features, restricted images), fall back to the standard toolchain for the lint job only, rather than blocking on it. Local dev and the Docker build keep the preferred one.

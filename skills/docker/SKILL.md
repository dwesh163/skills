---
name: docker
description: Rules for writing production Dockerfiles and .dockerignore files for any language or runtime — multi-stage builds, minimal non-root runtime images, pinned base images, cache-friendly layer order, no secret in any layer, and build-time placeholders for code that reads the environment at import time. Use when writing, reviewing or debugging a Dockerfile, containerizing an application, shrinking an image, or fixing a container build that fails in CI but passes locally.
metadata:
  tags: docker, dockerfile, containers, multi-stage-build, security, ci
---

# Dockerfile

These rules are language-agnostic. Apply them with the toolchain, package manager and runtime of the project at hand. If the repository already has a Dockerfile, keep its shape and fix what violates the rules below rather than rewriting it.

## Stage layout

Use a multi-stage build. Typical stages, each with one job:

1. **deps**: copy only the dependency manifest and lockfile, install dependencies from the lockfile (frozen / locked mode, never resolving new versions).
2. **builder**: copy the sources on top of the installed dependencies, generate code, compile or bundle.
3. **migrator** (only when the app owns a database): the migration tool and the migration files, nothing else.
4. **runner**: the smallest runtime image that can execute the build output.

Rules:

- The runner copies only what it executes: the build output, static assets, and the runtime dependencies that output needs. Never the sources, the full dependency tree when a pruned or bundled output exists, the build toolchain, tests, or any `.env`.
- Build on the runtime that will serve the output. If a fast tool installs dependencies but another runtime executes the app, compile in an image of the executing runtime and copy the installer binary in, so native bindings and generated code match production.
- Keep runtime and toolchain major versions identical across the Dockerfile, the local dev stack and CI. A build that only passes on one of them costs a day.
- Run migrations from their own stage, before the server starts, not from the application process.

## Images

- Pin every base image to an explicit version tag (major.minor at least), never `latest`. Let Dependabot or Renovate bump them (see the `github` skill).
- Prefer slim or distroless variants for the runner. Alpine only when the runtime and its native dependencies are known to work on musl.
- Use the same base image family across stages when native artifacts are copied between them.

## Security

- **Non-root.** Create a system user and group with a fixed uid/gid in the runner, `--chown` every `COPY` to it, and switch with `USER` before `CMD`.
- **No secret in any layer.** No `ENV` or `ARG` carrying a secret, no copied `.env`, no credential left in a file that a later `RUN` deletes (the earlier layer keeps it). Real values come from the container's environment at runtime.
- **Secrets needed during the build** (private registry token, private package index) go through BuildKit secret mounts, never through build args.
- **`.dockerignore` is a security file.** `COPY . .` sends everything not ignored into the build. Ignore at least: VCS metadata, CI configuration, dependency and build output directories, every `.env*` except the example file, local secrets files, the Dockerfile and compose files themselves, the Makefile, editor and OS files, and documentation not needed at build time.
- Set the runtime's production mode and disable telemetry through `ENV` in the runner.
- Expose only the port the app listens on, and bind the app to `0.0.0.0` inside the container.

## Code that reads the environment at import time

Builds often import or evaluate application modules (static generation, route collection, framework config). Any module that reads a required variable at load time and fails fast then breaks the CI build, while local builds pass because a local `.env` exists.

- Fix it in the Dockerfile: set a placeholder for each such variable **inline on the build command itself**, so it never persists in an image layer.
- List every variable read at import time. Keep that list in sync with the code (see the `dotenv` skill).
- Never paper over it in code with a fallback value: a misconfigured production deployment would then boot on the fake value.
- To reproduce a CI-only failure locally, run the production build with the variables unset and no `.env` present.

## Layer order and cache

- Copy manifests and lockfiles before the sources, so dependency installation is cached until the lockfile changes.
- Put rarely changing steps first and frequently changing steps last.
- Use BuildKit cache mounts for package-manager caches instead of leaving caches in layers.
- Never install dependencies in a lifecycle hook that needs source files the deps stage does not have yet (code generation belongs to the builder stage).

## Runtime

- `CMD` in exec form so the process receives signals. When a shell is needed to chain a migration and the server, end with `exec` so the server replaces the shell.
- One process per container. No process supervisor unless the image genuinely needs one.
- Add a `HEALTHCHECK` only when the orchestrator does not provide its own probes.

## Review checklist

- [ ] Multi-stage, runner contains only runtime artifacts
- [ ] Base images pinned, majors aligned with dev stack and CI
- [ ] Dependencies installed from the lockfile in a cached layer
- [ ] Non-root user with fixed uid, every `COPY` chowned
- [ ] No secret in `ENV`, `ARG`, copied files or any intermediate layer
- [ ] Import-time variables set as inline placeholders on the build command
- [ ] `.dockerignore` excludes VCS, CI, env files, secrets, build output
- [ ] Exec-form `CMD`, signals reach the app

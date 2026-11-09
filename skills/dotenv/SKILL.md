---
name: dotenv
description: Contract for application environment variables in any language — required variables fail at startup with the variable's name, optional ones carry their default at the point of use, no fallback for secrets, an `.env.example` that documents every name, and a checklist of every place a new variable must be added (code, example file, local env generation, Dockerfile build placeholders, deployment config). Use when adding, renaming, removing or reading an environment variable, writing configuration loading code, or debugging a missing or wrong configuration value.
metadata:
  tags: environment-variables, configuration, dotenv, secrets, twelve-factor
---

# Environment variables

## Reading variables

- **Required variables go through one helper** that reads the variable, throws an error naming it when it is missing or empty, and returns its value. Define it once per codebase, in the shared utilities module, in the project's language.
- **Call it at module scope or at startup**, in the module that needs the value, so the process refuses to boot and the error names the variable, instead of failing on the first request far from the cause.
- **Never silence a missing value**: no non-null assertion, no cast, no empty-string default. A missing variable then ends up as a literal `undefined`/`None`/empty string inside a URL or an authorization header.
- **Never a fallback for a secret or a URL.** A misconfigured production deployment would boot on the fake value.
- **Optional variables** are read plainly, with their default at the point of use. The required helper is only for what the app cannot run without.
- **Parse and validate once**: numbers, booleans, URLs and enums are converted and checked where they are read, not at each use.
- **Build-time public variables** (values a frontend bundler inlines into client code) usually must be referenced literally to be replaced, so they cannot go through a dynamic helper. Read them literally and pass them to the build as build arguments or CI environment, never as secrets.

## `.env.example`

- Lists every variable the app reads, required and optional.
- One comment per variable or group: what it is and where its value comes from.
- Placeholders only, never a real value, not even a development secret.
- Committed. Every other `.env*` file is gitignored (the ignore rule needs an explicit exception for the example file, since `.env*` also matches it).

## One variable, several places

When adding a variable, walk this list. When removing or renaming one, walk it again.

| Place | What goes there |
|---|---|
| The code | A call to the required helper, or a plain read with its default |
| `.env.example` | The name, a comment, a placeholder |
| Local env generation | The mapping from the shared secrets store, or the local value (see the `compose` skill) |
| Dockerfile | An inline placeholder on the build command, if the variable is read at import time (see the `docker` skill) |
| CI | The variable or secret, only if a CI job needs it |
| Deployment config | The runtime value: chart values, a secret reference, the platform's settings |

Each omission is loud by design: a name missing from local generation crashes the dev server with the variable's name, one missing from the Dockerfile fails the CI build, one missing from deployment fails the rollout. A forgotten place is found by a developer or the pipeline, not by a user.

## Secrets

- Generated per project and per environment, never copied from another project's `.env`.
- Per-developer secrets (session signing keys and the like) are generated locally with a cryptographic random source, and kept across regenerations of the local `.env` so sessions survive.
- Shared development secrets come from the team's secret store, never from chat, email or a committed file.
- Values that end up inside a connection URL must be URL-safe or percent-encoded.

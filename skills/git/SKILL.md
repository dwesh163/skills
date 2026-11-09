---
name: git
description: Commit message format and commit granularity for any repository, whatever the language. Writes `[tag] Capitalized imperative description` messages (not Conventional Commits), keeps version bumps in their own commit, and splits work into small coherent commits so generated release notes stay readable. Use when writing, reviewing or rewording a commit message, choosing a commit tag, splitting staged changes into commits, or bumping a project's version.
metadata:
  tags: git, commits, commit-message, changelog, release-notes, semver
---

# Git commits

## Message format

`[tag] Capitalized imperative description`

- Tag in lowercase, inside square brackets, then one space.
- Description starts with a capital letter, imperative mood, no trailing period.
- Subject line only, unless the *why* is not obvious from the diff: then a blank line and a short body explaining the reason, not restating the change.
- Never Conventional Commits syntax (`type:` or `type(scope):`). The release-notes generator matches `[tag]`, so a colon-style commit lands in "Other Changes".

## Tags

Use one of these. Each maps to a section of the generated release notes:

| Tag | For | Release-notes section |
|---|---|---|
| `add` | New capability, new file, new option | Features |
| `feature` | A larger user-facing capability | Features |
| `fix` | Bug fix | Bug Fixes |
| `update` | Change to existing behavior that is neither a fix nor a refactor | Maintenance |
| `refactor` | Behavior-preserving code change | Maintenance |
| `chore` | Dependencies, configuration, CI, tooling, housekeeping | Maintenance |
| `version` | The version-bump commit, nothing else | Maintenance |
| `docs` | Documentation only | Documentation |

Pick the tag from what the commit *does*, not from the branch or ticket it belongs to. If two tags both seem true, the commit probably holds two changes: split it.

## Small commits

Build a change as a sequence of small commits, each one coherent and leaving the project working, each tagged on its own merits. Do not squash a naturally multi-step change (schema, then logic, then interface, then wiring) into a single commit.

- Every commit becomes one bullet in the release notes: one large commit becomes one vague bullet.
- Tag counts stay meaningful instead of everything defaulting to the one tag that was true for part of a large commit.
- Do not fragment unrelated edits artificially either: one logical step per commit.

When splitting staged work, stage by hunk or by file, and check that each intermediate commit still builds.

## Version bumps

- A version bump is its own `[version]` commit, touching only the version field of the project's manifest (or its version file). Nothing else in the diff.
- Subject: `[version] Bump to <x.y.z> in <manifest file>`.
- Semver: patch for fixes and small refactors, minor for backward-compatible features, major for breaking changes.
- Bump once a batch of merged work is ready to ship, not on every commit. When the release pipeline releases on version change (see the `github` skill), pushing the `[version]` commit to the main branch *is* the release.

## Before committing

- Read the diff that is actually staged, not the one you remember writing.
- No secret, `.env` file, generated artifact or local configuration in the diff.
- If the repository documents its own commit convention (`CONTRIBUTING.md`, `CLAUDE.md`, `AGENTS.md`), that convention wins.

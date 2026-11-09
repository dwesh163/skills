---
name: nextjs
description: House conventions for Next.js App Router apps in TypeScript with Bun, Biome, Tailwind and shadcn/ui, Prisma, better-auth, next-intl and zod. Covers file organization, auth and the proxy.ts route guard, the service layer and typed errors, server vs. client components, validation, translations, permissions, Stripe billing, UI theming, SEO, legal pages, and the Next.js-specific Dockerfile, CI steps and local dev stack. Use when scaffolding a Next.js project or writing or reviewing code in one that follows this stack.
metadata:
  tags: nextjs, react, typescript, app-router, bun, biome, prisma, better-auth, next-intl, zod, shadcn-ui, tailwind
---

# Next.js conventions

## When to use

Load this skill whenever you're touching a Next.js (App Router) project that
matches this stack: TypeScript, Bun, Biome (not ESLint/Prettier), Tailwind +
shadcn/ui, `next-intl`, `better-auth`, Prisma, `zod`. Also use it when asked
to scaffold a new Next.js project from scratch — these are the conventions to
reproduce.

These conventions are distilled from a range of production Next.js apps, from
small internal tools to full multi-domain platforms, plus a scaffolding CLI
that bootstraps new projects into this same shape. They're opinionated and
consistent by default — match the existing shape in whatever repo you're in
rather than inventing a new one.

Not every app needs every piece described here. Two dimensions vary the most
from project to project, and each reference file below says so explicitly
where it matters:

- **Does the app own its database, or proxy another service?** A Prisma-backed
  app and a thin BFF over an external REST API need a different service-layer
  shape — see services-and-errors.md.
- **Where do permissions come from — an IdP's groups, or something local?**
  Group/role claims from an OIDC provider and app-local or delegated
  authorization need different wiring — see permissions.md.

Pick the case that matches the project you're in; don't import machinery
(better-auth's access-control plugin, `ServiceResponse`, a database at all)
that a smaller app doesn't need.

In an existing repo, the code itself usually answers which case applies —
read it, don't ask what's already on disk. But if the repo is ambiguous or
only partially set up (e.g. no Prisma schema yet but unclear if that's
intentional, or an i18n folder with only one locale ever added to it), don't
guess which way to extend it — ask directly, same as the fresh-scaffold case
below.

Scaffolding a fresh project is the clearer version of the same problem:
there's no code to read at all, so don't silently default to the full stack
table below. Ask the user first, as a short batch of questions rather than
one at a time:

- **Database** — does it own data via Prisma, or is it a thin BFF/proxy with
  no local DB (services-and-errors.md's Shape B, see prisma.md)?
- **i18n** — multiple locales via `next-intl`, or single-language with no
  translation layer at all (see translations.md)?
- **Auth** — needed at all, and if so DB-backed vs stateless sessions, which
  OIDC provider (see auth.md)?
- **Permissions** — IdP groups, app-local/delegated roles, org-scoped
  multi-tenant, or nothing beyond "logged in" (see permissions.md)?
- **Billing** — `@better-auth/stripe`, per-user or per-organization, or not
  needed (see stripe.md)?

Only wire in what the user actually confirms — this is the same
"don't import machinery a smaller app doesn't need" rule above, applied
before any code exists instead of after.

If a repo has its own `rules/`, `CLAUDE.md` or `AGENTS.md`, that repo's
version wins over this skill where the two disagree; those are living
documents for that specific codebase.

## Stack at a glance

| Concern | Choice |
|---|---|
| Framework | Next.js 16, App Router, TypeScript, `src/` dir, `@/*` alias |
| Package manager | Bun (`bun.lock`, `bunx`) |
| Lint/format | Biome (`biome.json`) — no ESLint, no Prettier |
| Styling | Tailwind v4 + shadcn/ui (`components.json`, `style: new-york`), `cva`, `next-themes` — see ui.md |
| Auth | `better-auth`, DB-backed sessions, OIDC provider (Entra ID: identity from the verified ID token), `src/proxy.ts` guard |
| Env vars | every required server-side variable read through `required("NAME")`, never `process.env.X!` — see tooling.md and the `dotenv` skill |
| i18n | `next-intl`, `en`/`fr`, cookie + `Accept-Language` detection |
| Database | Prisma (7.x), `pg` adapter, schema at `src/prisma/schema.prisma`, snake_case in Postgres via `@map`/`@@map` on everything |
| Validation | `zod` v4, schema-factory pattern for localized messages |
| Permissions | IdP groups → better-auth's `admin` plugin + access control, or app-local/delegated — see permissions.md |
| Forms | `react-hook-form` + `@hookform/resolvers/zod` |
| Containers | Multi-stage `Dockerfile` (Bun installs, Node builds and runs), `output: "standalone"`, non-root — rules in the `docker` skill |
| CI/CD | Reusable lint/build workflows, release on version bump, hardened — rules in the `github` skill |
| Commits | `[tag] Capitalized description`, not Conventional Commits — the `git` skill |
| Local dev | `make up` — Postgres + the Next.js dev server in Docker Compose, `.env` from a shared `secrets.yml` — see local-dev.md and the `compose` skill |
| Billing (optional) | `@better-auth/stripe`, per-user or per-organization — see stripe.md |
| Legal pages (optional) | static `/legal/<slug>` routes, one shared component, next-intl content — see legal.md |
| SEO (optional) | file-based metadata: title template, OG images, `sitemap.ts`, `robots.ts`, `noindex` for private routes — see seo.md |

## Reference files

Read the matching file before touching that area of a codebase:

- [references/file-organization.md](references/file-organization.md) — `src/` layout, domain mirroring across `types`/`validations`/`services`/`components`, the `providers/` folder, generic cross-domain component folders (`dialog/`, ...), naming conventions
- [references/auth.md](references/auth.md) — `better-auth` setup, OIDC providers (Entra ID / ID-token claims), DB-backed vs stateless sessions, `proxy.ts` route guard, token refresh, `getUser()`/`session()`, optional plugins (two-factor/TOTP)
- [references/services-and-errors.md](references/services-and-errors.md) — the service layer (`ServiceResponse<T>`/`toResponse` vs. thin REST-BFF), the `ExpectedError` hierarchy, the not-found/error-boundary trio, and `load()` for reads whose failure must be shown rather than swallowed
- [references/client-components.md](references/client-components.md) — server vs. `"use client"` boundary, `<Fragment>` instead of `<>...</>`, data-fetching split, server actions, and when a real `api/.../route.ts` is still justified over a server function
- [references/validation.md](references/validation.md) — zod schema-factory pattern, shared `common.ts` schemas, wiring into `react-hook-form`
- [references/translations.md](references/translations.md) — `next-intl` message schema, ICU rules, gender handling, component-side conventions
- [references/prisma.md](references/prisma.md) — schema location, `prisma.config.ts`, snake_case naming with `@map`/`@@map` (and renaming a live database), client singleton, generated-client placement
- [references/permissions.md](references/permissions.md) — better-auth's access control for IdP-groups permissions vs. app-local/delegated authorization vs. org-scoped multi-tenant roles, route-level checks, when to reach for which
- [references/ui.md](references/ui.md) — `components.json`, adding/customizing shadcn components, `cva` variants, `cn()`, light/dark theming (CSS variables, `next-themes`, the theme toggle)
- [references/tooling.md](references/tooling.md) — Bun scripts, Biome config, `next.config.ts`, root layout wiring, the Next.js Dockerfile and the env `next build` needs at import time, the stack-specific CI steps, `required()`
- [references/local-dev.md](references/local-dev.md) — the concrete `Makefile` (with its Keybase-sourced `.env`) and `docker-compose.dev.yml` for Postgres + the Next.js dev server
- [references/stripe.md](references/stripe.md) — `@better-auth/stripe` billing, per-user vs per-organization subscriptions, the `authorizeReference` permission check, webhook routing
- [references/legal.md](references/legal.md) — privacy policy/terms-of-use routes, the shared `LegalDocument` component, `LEGAL_LAST_UPDATED`, keeping the routes public in `proxy.ts`
- [references/seo.md](references/seo.md) — `SITE` constant, root/per-page metadata and title template, Open Graph images, `sitemap.ts`, `robots.ts`, icons, `noindex` for private routes, JSON-LD, the i18n caveat

## Related skills

Language-agnostic rules this skill builds on, each its own skill in the same
collection: `git` (commit format, version bumps), `docker`,
`github` (GitHub Actions / GitLab CI pipeline and hardening),
`compose` (Docker Compose dev stack, Makefile, `.env` from shared
secrets), `dotenv`. The reference files above only hold what
is specific to Next.js.

These are living documents: when a convention stops holding in a repo, or a
repo evolves one, update the matching file instead of silently diverging.

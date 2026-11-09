# Tooling: Biome, next.config, root layout, Dockerfile, CI, env

## Contents

- Package manager & scripts
- Biome — not ESLint, not Prettier
- shadcn/ui and theming
- `next.config.ts`
- Root layout wiring order
- Dockerfile — multi-stage, Bun installs, Node builds and runs, standalone output
- CI — what is specific to this stack
- Environment variables — `required()`, `.env.example`, and keeping the lists in sync

Next.js-specific instances of the generic rules in the `docker`, `github` and
`dotenv` skills. Those skills hold the rules; this file holds what
they look like on this stack.

## Package manager & scripts

Bun, not npm/pnpm/yarn — `bun.lock` committed, `bunx` for one-off runners.
Standard scripts:

```json
"scripts": {
  "dev": "next dev",
  "build": "next build",
  "start": "next start",
  "lint": "biome check",
  "format": "biome format --write"
}
```

(`dev`/`build` get prefixed with Prisma generation when the Prisma module is
present — see prisma.md.)

## Biome — not ESLint, not Prettier

```json
{
  "$schema": "https://biomejs.dev/schemas/2.4.6/schema.json",
  "vcs": { "enabled": true, "clientKind": "git", "useIgnoreFile": true },
  "files": { "ignoreUnknown": true, "includes": ["**", "!src/components/ui", "!node_modules", "!.next"] },
  "formatter": { "enabled": true, "indentStyle": "space", "indentWidth": 2, "lineWidth": 120 },
  "linter": { "enabled": true, "rules": { "recommended": true, "suspicious": { "noUnknownAtRules": "off" } } },
  "javascript": { "formatter": { "quoteStyle": "double", "trailingCommas": "all", "semicolons": "always" } },
  "css": { "parser": { "cssModules": true, "tailwindDirectives": true }, "formatter": { "enabled": true } },
  "assist": { "enabled": true, "actions": { "source": { "organizeImports": "on" } } }
}
```

`src/components/ui` (shadcn-generated primitives) is always excluded from
linting — it's generated code, don't fight its style or "fix" it to match
Biome's rules. `noUnknownAtRules: "off"` accommodates Tailwind's `@apply`/
`@theme` at-rules. Double quotes, trailing commas, semicolons always, 2-space
indent, 120-col lines — don't introduce Prettier or an ESLint config
alongside this.

CI runs `bunx biome ci --reporter=github ./src` (fails the build) and
separately `bunx biome lint --reporter=json ./src` uploaded as an artifact
for review — see the GitHub Actions section below.

## shadcn/ui and theming

`components.json`, adding/customizing components, `cn()`, and the
light/dark theme system (CSS variables, `next-themes`, the root layout's
`ThemeProvider`) have their own file — see ui.md.

## `next.config.ts`

```ts
import type { NextConfig } from "next";
import createNextIntlPlugin from "next-intl/plugin";

const withNextIntl = createNextIntlPlugin("./src/i18n/request.ts");

const nextConfig: NextConfig = {
  output: "standalone", // needed for the Docker runner stage, see below
  outputFileTracingRoot: __dirname,
};

export default withNextIntl(nextConfig);
```

`serverActions.bodySizeLimit` gets bumped in the `experimental` block for
apps that accept large file uploads through a server action. `images.remotePatterns`
gets an explicit allowlist per external image host actually used — don't use
a wildcard hostname.

## Root layout wiring order

better-auth needs no provider in the root layout at all (its client is
store-backed — see auth.md), so the layout stays about as thin as this
stack gets. `NextIntlClientProvider` wraps everything (it's what makes
translations available to every client component below it); `TooltipProvider`
(shadcn) wraps `children` for hover-card/tooltip primitives used anywhere in
the tree; `Toaster` is a sibling, not a wrapper, since it renders its own
portal rather than providing context. Any app-specific `providers/*` wrapper
(see file-organization.md) nests between those two, outer to inner in
whatever order one depends on another:

```tsx
export const dynamic = "force-dynamic";

export default async function RootLayout({ children }: { children: React.ReactNode }) {
  const locale = await getLocale();
  const messages = await getMessages();

  return (
    <html lang={locale} className={cn(inter.variable, localFont.variable)}>
      <body className="antialiased font-sans">
        <NextIntlClientProvider messages={messages}>
          <TooltipProvider>{children}</TooltipProvider>
          <Toaster position="bottom-center" />
        </NextIntlClientProvider>
      </body>
    </html>
  );
}
```

`export const dynamic = "force-dynamic";` — `getLocale()` reads a cookie/the
`Accept-Language` header on every request (see translations.md), so static
optimization of the root layout isn't applicable; declare it explicitly
rather than relying on Next's automatic detection.

This is the baseline. An app with a light/dark theme toggle adds one more
outermost wrapper (`ThemeProvider` from `next-themes`) plus
`suppressHydrationWarning` on `<html>` — see ui.md, which shows the full
layout with that addition rather than repeating it here.

## Dockerfile — multi-stage, Bun installs, Node builds and runs, standalone output

```dockerfile
# Bun installs; Node builds and runs the standalone server (the Bun image has no `node`).
FROM oven/bun:1.4-slim AS deps
WORKDIR /app
COPY package.json bun.lock ./
RUN bun install --frozen-lockfile

# Node with Bun copied in: Bun drives the scripts, Next builds on the runtime it will run on.
FROM node:26-slim AS builder
COPY --from=oven/bun:1.4-slim /usr/local/bin/bun /usr/local/bin/bun
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1
# Placeholders so the modules that read the environment at import time (better-auth, API clients,
# Prisma) can be evaluated while Next collects page data. Inline on these commands, so they never
# reach an image layer; the real values come from the container's environment at runtime.
RUN DATABASE_URL=postgresql://build@localhost/build bun run prisma generate
RUN AUTH_SECRET=build-time-placeholder \
    AUTH_URL=http://localhost:3000 \
    CLIENT_ID=build CLIENT_SECRET=build \
    IDENTITY_ISSUER=http://localhost \
    DATABASE_URL=postgresql://build@localhost/build \
    node node_modules/.bin/next build

# Prisma only. The standalone output ships just what the server imports, so the Prisma CLI and the
# migrations live in their own stage; `migrate deploy` runs before the server starts.
FROM node:26-slim AS migrator
WORKDIR /opt/migrations
COPY --from=builder /app/package.json /tmp/package.json
RUN npm install --no-save --no-package-lock --omit=dev \
      "prisma@$(node -p "require('/tmp/package.json').devDependencies.prisma")" \
      "dotenv@$(node -p "require('/tmp/package.json').devDependencies.dotenv")"
COPY prisma.config.ts ./
COPY src/prisma ./src/prisma

FROM node:26-slim AS runner
WORKDIR /app
RUN groupadd --system --gid 1001 nodejs && useradd --system --uid 1001 --gid nodejs nextjs
ENV NODE_ENV=production NEXT_TELEMETRY_DISABLED=1 PORT=3000 HOSTNAME="0.0.0.0"
COPY --from=migrator --chown=nextjs:nodejs /opt/migrations /opt/migrations
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
COPY --from=builder --chown=nextjs:nodejs /app/public ./public
USER nextjs
EXPOSE 3000
CMD ["sh", "-c", "cd /opt/migrations && node node_modules/prisma/build/index.js migrate deploy && cd /app && exec node server.js"]
```

Without Prisma: drop the `prisma generate` line, the whole `migrator` stage, its `COPY`, and the
`CMD` becomes `["node", "server.js"]`.

- **Why Node in the builder and not the Bun image.** `next build` runs on the runtime that will
  serve the output, so what it bakes in (native bindings, Node-specific code paths) matches
  production. Bun is only copied in as the script runner (`bun run prisma generate`). The final
  `next build` is invoked as `node node_modules/.bin/next build`, not `bunx next build` (no Bun
  re-resolving in a stage that already has `node_modules`).
- **Why the runner is Node.** The Bun image ships no `node` binary, so `CMD ["node", "server.js"]`
  fails on it. `deps` stays Bun, only the runtime needs Node.
- **Non-root, fixed uid.** The runner runs as `nextjs` (1001) and every `COPY` is `--chown`ed to it.
  It copies only the `standalone` output (requires `output: "standalone"` in `next.config.ts`), the
  static assets and `public/` — never the full `node_modules`, the sources, or any `.env`.
- **Keep the Node and Bun majors identical** across the Dockerfile, `docker-compose.dev.yml` and CI
  (`setup-bun`, `setup-node`): a build that passes on one runtime and fails on another is a waste of
  a day.
- **`.dockerignore`**: apply the `docker` skill's list, plus this stack's own output
  directories (`.next`, `node_modules`, the generated Prisma client under `generated`).

### Env that `next build` needs at import time (better-auth and friends)

`next build` evaluates server modules while it collects page data, **with `NODE_ENV=production`
and none of the runtime environment** (the Docker `builder` stage has no secrets, and it must not
get any). Anything that reads `process.env` at import time and throws on a missing value therefore
fails the GitHub build, while `next dev` and a local build with a `.env` pass:

```
[Error [BetterAuthError]: You are using the default secret. Please set `BETTER_AUTH_SECRET` ...]
WARN [Better Auth]: Base URL is not set. ...
```

better-auth is the usual culprit (`betterAuth({...})` at module scope refuses to load in
production without a secret), and so is every `required("NAME")` (see the last section of this file)
that runs at module scope. Fix it in the `Dockerfile`, with placeholders set **inline on the build
command** — not `ENV`/`ARG`, which would persist in the image layers; the real values come from the
container's environment when it starts. The `builder` stage of the Dockerfile above shows the shape:
one `NAME=placeholder` per variable, on the `RUN ... next build` line.

List every variable that the auth instance, `lib/api.ts` and any other module-scope `required()`
read. Don't paper over it in code (`secret ?? "x"`) — that would also let a misconfigured production
deployment boot with a fake secret. Discovery fetches against the placeholder issuer (`discoveryUrl`)
fail during the build with a logged `Discovery fetch failed` / "Provider skipped" error: expected and
harmless there, the provider is registered for real at runtime. To reproduce the CI failure locally,
run `NODE_ENV=production next build` with the variables **unset** (no `.env`).

## CI — what is specific to this stack

The pipeline shape, the four workflow files and the security baseline come from the `github`
skill. On this stack:

- **`_lint.yaml` steps**: checkout (`persist-credentials: false`), `oven-sh/setup-bun` with the
  same Bun major as the Dockerfile, `bun install --frozen-lockfile`, `bunx prisma generate` with a
  placeholder `DATABASE_URL` (only with Prisma), `bunx biome ci --reporter=github ./src` (fails the
  build), `bun run typecheck`, then `bunx biome lint --reporter=json ./src > biome-report.json || true`
  uploaded as a non-failing artifact with `if: always()`.
- **`_build.yaml`** writes the input version into `package.json` with `jq` (version passed through
  `env:`) before the Docker build, so the image reports the version it was built for.
- **`detect-version`** runs with `type: js`: the version is read from `package.json`.
- **Dependabot** adds the `npm` ecosystem (Bun's lockfile is supported) next to `github-actions` and
  `docker`.
- **If CI runners cannot run Bun** (old or restricted runner images missing CPU features Bun's
  baseline build needs), the lint job alone falls back to `node`/`npm`; local dev and the Docker
  build stay on Bun.

## Environment variables — `required()`, `.env.example`, and keeping the lists in sync

Every server-side variable the app cannot run without is read through one helper, next to `cn()`:

```ts
// src/lib/utils.ts
// An environment variable that must be set: fails at startup, not on the first request.
export function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing environment variable ${name}`);
  return value;
}
```

Call it at **module scope** of the module that needs the value, so the process refuses to boot and
the message names the variable:

```ts
const CLIENT_ID = required("CLIENT_ID");

export const auth = betterAuth({
  secret: required("AUTH_SECRET"),
  baseURL: required("AUTH_URL"),
  // ...
});
```

- **Never `process.env.X!`.** The `!` only silences the compiler: a missing variable becomes the
  string `"undefined"` inside a URL or a `Bearer` header and fails far from the cause. Never
  `?? "fallback"` for a secret or a URL either — a misconfigured production would boot on the fake
  value (see "Env that `next build` needs at import time" above).
- **Optional variables** are read plainly, with the default at the point of use
  (`process.env.LOG_LEVEL ?? "info"`). `required()` is only for what the app cannot run without.
- **`NEXT_PUBLIC_*` cannot go through it.** Next inlines `process.env.NEXT_PUBLIC_X` into the client
  bundle only when it is written literally; a dynamic `process.env[name]` is not replaced, so it
  would be `undefined` in the browser. Read those literally (they are fixed at build time, so they
  reach the build as Docker build args / CI env).
- **It runs during `next build`.** Next imports server modules while collecting page data, with none
  of the runtime environment, so a module-scope `required()` throws there too — hence the inline
  placeholders on the Dockerfile's `next build` line.

**One variable lives in several places** (code, `.env.example`, the Makefile's `SECRETS` or
`LOCAL_VARS`, a placeholder on the Dockerfile's `next build` line, the deployment config): walk the
checklist in the `dotenv` skill whenever one is added, renamed or removed.

`AUTH_SECRET` is generated per project and per developer (`openssl rand -base64 32`, done by
`make env`), never copied from another project's `.env`.

Apps with public pages also declare `NEXT_PUBLIC_APP_NAME` and `NEXT_PUBLIC_APP_URL` (canonical
origin, no trailing slash — both inlined at build time, so pass them as Docker build args / CI env)
and optionally `ROBOTS_INDEX=false` for staging; both are used by seo.md (metadata, sitemap,
robots).

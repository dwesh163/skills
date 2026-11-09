# Authentication

## Contents

- `src/lib/auth/server.ts` — the server instance
- Variant: a built-in social provider instead of `genericOAuth`
- Entra ID, or any OIDC IdP whose claims are in the ID token — the worked case
- Two ways to persist a session: DB-backed (default) vs stateless
- Ory Hydra, stateless (no database) — the worked case
- `src/proxy.ts` — the route guard
- `src/app/api/auth/[...all]/route.ts` — the one route handler
- `src/lib/auth/client.ts` — the client instance, no provider needed
- Token refresh
- Optional plugin: two-factor / TOTP
- Optional plugin: Stripe billing (`@better-auth/stripe`)
- Never bypass `getUser()`/`session()`

`better-auth`, its Prisma adapter, a single OIDC provider, and a route guard
in `src/proxy.ts`. Session state lives in the database (a real `session`
table via the adapter), not a signed JWT cookie — better-auth manages the
`user`/`session`/`account`/`verification` tables itself (see prisma.md on
generating that part of the schema).

> better-auth's plugin API moves fast. The shape below (`genericOAuth`,
> `admin` plugin, `createAccessControl`) is accurate as of writing — check
> the installed version's docs for exact option names before treating any
> single field name as gospel; the overall architecture (adapter, plugins,
> `auth.api.*` server calls, `authClient.*` client calls) is stable.

## `src/lib/auth/server.ts` — the server instance

`auth/` is a folder, not `lib/auth.ts` + `lib/auth-client.ts` — "auth" is a
real category with two files in it (a server instance and a client
instance), so it nests instead of getting hyphenated apart, same rule as
file-organization.md's "nest instead of hyphenating" (`session-provider.tsx`
→ `providers/session.tsx` is the same move).

```ts
import { betterAuth } from "better-auth";
import { prismaAdapter } from "better-auth/adapters/prisma";
import { genericOAuth } from "better-auth/plugins";
import { prisma } from "@/lib/prisma";
import { required } from "@/lib/utils";

export const auth = betterAuth({
  secret: required("AUTH_SECRET"),
  baseURL: required("AUTH_URL"),
  database: prismaAdapter(prisma, { provider: "postgresql" }),
  plugins: [
    genericOAuth({
      config: [
        {
          providerId: "identity-provider",
          clientId: required("CLIENT_ID"),
          clientSecret: required("CLIENT_SECRET"),
          discoveryUrl: `${required("IDENTITY_ISSUER")}/.well-known/openid-configuration`,
          scopes: ["openid", "email", "profile"],
          // Extra IdP claims (group membership, an internal username) that don't map to
          // better-auth's default user fields land here, at account-link time.
          mapProfileToUser: (profile) => ({
            username: profile.preferred_username ?? profile.sub,
            groups: profile.groups ?? [],
          }),
        },
      ],
    }),
  ],
  user: {
    additionalFields: {
      username: { type: "string", required: false },
      groups: { type: "string[]", required: false },
    },
  },
});

export async function session() {
  const { headers } = await import("next/headers");
  return auth.api.getSession({ headers: await headers() });
}

export async function getUser() {
  const result = await session();
  if (!result?.user) throw new Error("User not authenticated"); // or UnauthorizedError, see services-and-errors.md
  return result.user;
}
```

Every environment variable is read through `required("NAME")`, never `process.env.NAME!` — the
process then refuses to start with a message naming the missing variable, instead of failing on the
first sign-in with `undefined` in a URL. See tooling.md ("Environment variables") for the helper,
the `next build` placeholders it implies, and the places a new variable has to be added.

`user.additionalFields` is what makes `username`/`groups` (and whatever else
a given app needs) show up, fully typed, on `session.user` everywhere —
there's no manual module-augmentation step like the old `next-auth.d.ts`
pattern; types are inferred straight from this config
(`typeof auth.$Infer.Session`).

Named `session()`, not `auth()` — `auth` is already taken two lines up by
the betterAuth instance itself (`export const auth = betterAuth(...)`), so
a wrapper function can't reuse that name in the same module; this is the
same class of naming collision as `organisations`/`organisations.list()` in
services-and-errors.md, and the same fix (pick a name that isn't already
bound, don't fight the shadow). `session()` mirrors better-auth's own
`getSession` naming and returns the raw `{ session, user } | null` — the
honest, unopinionated primitive. `getUser()` is the convenience layer most
call sites actually want (the current user or a thrown error, no null
check at every call site) — built on top of `session()`, not a duplicate
implementation. Reach for `session()` directly only when `null` is a
meaningful, handled case (an optional sign-in state on a public page), not
a caught exception.

## Variant: a built-in social provider instead of `genericOAuth`

better-auth ships first-class providers for the common IdPs (`socialProviders.microsoft`
for Entra ID/Azure AD among others) that need less config than the generic
OIDC plugin, at the cost of less control over exactly which OIDC endpoint/
scope set gets used:

```ts
export const auth = betterAuth({
  database: prismaAdapter(prisma, { provider: "postgresql" }),
  socialProviders: {
    microsoft: {
      clientId: required("ENTRA_ID"),
      clientSecret: required("ENTRA_SECRET"),
      tenantId: required("ENTRA_TENANT_ID"),
    },
  },
});
```

Reach for `genericOAuth` (the main example above) for any IdP without a
built-in provider — a self-hosted Ory Hydra/Keycloak instance, or any
standards-compliant OIDC issuer identified only by its discovery URL. Reach
for a built-in `socialProviders` entry when the IdP has one and its default
scope/claim set is already enough; it's less config for the same result.
Both populate `session.user` the same way once `mapProfileToUser` (generic)
or the provider's own defaults (built-in) are configured — nothing else
downstream (`getUser()`, `proxy.ts`, services) needs to know which one is in
use.

**Entra ID is the exception to "less config is better":** as soon as the app needs a claim beyond
name and email (a stable user id, groups, a role), take the `genericOAuth` route on the tenant's
issuer so the ID token is verified and `mapProfileToUser` sees its claims — that is the next section.

## Entra ID, or any OIDC IdP whose claims are in the ID token — the worked case

Use this when what the app keys on — a stable user id, group membership, a role — is a **claim in
the ID token**. That is the normal case with Microsoft Entra ID (and Keycloak, or Hydra with a token
hook). Use `genericOAuth` against the issuer's discovery document and let better-auth verify and
decode the ID token: `mapProfileToUser` then receives *verified* claims, with no userinfo or Graph
round-trip. The built-in `socialProviders.microsoft` above is enough only when name and email are all
you need.

```ts
// src/lib/auth/provider.ts — no server import, so client code and the proxy can use it too
// The redirect URI registered on the IdP app is <AUTH_URL>/api/auth/callback/<PROVIDER_ID>.
export const PROVIDER_ID = "entra-id";

// Where a failed sign-in lands. NOT behind proxy.ts: sending the visitor back to the IdP from here
// would loop forever on a persistent failure.
export const AUTH_ERROR_PATH = "/denied";
```

```ts
// src/lib/auth/server.ts
import "server-only";
import { betterAuth } from "better-auth";
import { prismaAdapter } from "better-auth/adapters/prisma";
import { APIError } from "better-auth/api";
import { nextCookies } from "better-auth/next-js";
import { genericOAuth } from "better-auth/plugins/generic-oauth";
import { AUTH_ERROR_PATH, PROVIDER_ID } from "@/lib/auth/provider";
import { prisma } from "@/lib/prisma";
import { required } from "@/lib/utils";

const SESSION_SECONDS = 60 * 60 * 8;

export const auth = betterAuth({
  appName: "<App name>",
  secret: required("AUTH_SECRET"),
  baseURL: required("AUTH_URL"),
  database: prismaAdapter(prisma, { provider: "postgresql" }),
  // Behind a reverse proxy / router the protocol is mis-detected without this (http instead of
  // https): cookies are mis-set or dropped and the sign-in restarts on every request.
  advanced: { trustedProxyHeaders: true },
  session: { expiresIn: SESSION_SECONDS, cookieCache: { enabled: true, maxAge: 5 * 60 } },
  account: {
    encryptOAuthTokens: true, // the tokens stored in the account table are encrypted at rest
    accountLinking: { enabled: false }, // one IdP, one identity: never merge accounts by email
  },
  user: {
    additionalFields: {
      // `input: true` is what lets mapProfileToUser fill them. /update-user is disabled below, so
      // only the IdP ever writes them.
      uniqueId: { type: "string", required: false, input: true, unique: true },
      groups: { type: "string[]", required: true, defaultValue: [], input: true },
    },
  },
  // Without this, production redirects API errors to `/`, which would restart the sign-in flow.
  onAPIError: { errorURL: AUTH_ERROR_PATH },
  // Nothing about a user is editable from the browser: identity comes from the IdP only.
  disabledPaths: ["/update-user", "/change-email", "/change-password", "/delete-user", "/set-password"],
  plugins: [
    genericOAuth({
      config: [
        {
          providerId: PROVIDER_ID,
          clientId: required("CLIENT_ID"),
          clientSecret: required("CLIENT_SECRET"),
          // Entra: https://login.microsoftonline.com/<tenant-id>/v2.0
          discoveryUrl: `${required("IDENTITY_ISSUER")}/.well-known/openid-configuration`,
          requireIdTokenVerification: true,
          pkce: true,
          scopes: ["openid", "profile", "email"],
          // Refresh the id and the groups from the IdP at every sign-in, not only the first one.
          overrideUserInfo: true,
          // Signing out ends the app session only; the IdP's SSO session stays.
          disableProviderLogout: true,
          // `profile` is the ID token: verified (signature, issuer, audience, expiry, nonce) and
          // decoded by better-auth.
          mapProfileToUser: (profile) => {
            // Entra's stable user id is `oid`; `sub` is pairwise, a different value in every app.
            if (typeof profile.oid !== "string" || !profile.oid)
              throw new APIError("FORBIDDEN", { message: "The identity provider sent no oid" });
            return {
              name: [profile.given_name, profile.family_name].filter(Boolean).join(" ") || profile.name,
              emailVerified: profile.emailVerified ?? true, // single tenant: it owns its addresses
              uniqueId: profile.oid,
              groups: Array.isArray(profile.groups) ? profile.groups : [],
            };
          },
        },
      ],
    }),
    nextCookies(), // last: lets server actions set cookies
  ],
});
```

What each non-obvious line is for:

- **`requireIdTokenVerification: true` — the one that matters when identity comes from the ID
  token.** Without it, a discovery document that is unreachable or lacks the issuer/JWKS silently
  degrades the provider to decoding the ID token *without verifying it*. With it, the provider is
  not registered on incomplete discovery and sign-in fails instead of trusting an unverified token.
  (At build time against the placeholder issuer the same condition logs `Discovery fetch failed` —
  expected, see tooling.md.)
- **`IDENTITY_ISSUER` is the tenant-specific issuer** (`https://login.microsoftonline.com/<tenant-id>/v2.0`),
  not `common` / `organizations`: those discovery documents carry a `{tenantid}` template in
  `issuer`, which never equals the issuer of a real token, so verification cannot succeed.
- **The provider id is part of the redirect URI.** Register `<AUTH_URL>/api/auth/callback/<PROVIDER_ID>`
  on the app registration (platform *Web*), one per environment (`http://localhost:3000/...` for
  dev), matching scheme, host, port and path exactly. Changing `PROVIDER_ID` later means
  re-registering.
- **Claims are configured at the IdP, not in better-auth.** `mapProfileToUser` only reads what the
  ID token carries: add the `groups` claim (and optional claims such as `given_name`) under *Token
  configuration* on the Entra app registration. A user in more than 200 groups gets no `groups`
  claim at all, only a `_claim_names` pointer to Graph; emit just the groups *assigned to the
  application* (`groupMembershipClaims: ApplicationGroup`) if users can be in that many.
- **`overrideUserInfo: true`** (default `false`) rewrites the user from the provider at every
  sign-in. Without it the groups are written once, at the first sign-in, and a group change never
  reaches the app. The role derived from them — in `databaseHooks.user.create/update.before` — is
  permissions.md's job ("Mapping IdP groups onto a role at sign-in").
- **`input: true` on the additional fields**, or the values `mapProfileToUser` returns are dropped
  (and a `required` field then fails the sign-in with `MISSING_FIELD`).
- **Throw in `mapProfileToUser` when the claim you key on is missing.** A user without it cannot use
  the app; failing the sign-in beats creating an identity-less row.
- **`accountLinking: { enabled: false }`** — never merge two sign-ins by email address.
- **`pkce: true`** even for a confidential client: it costs nothing and closes the code-interception
  case.
- **`disableProviderLogout: true`** keeps `signOut()` local: only the app session ends, and the next
  visit signs the user straight back in while their IdP session lives. A real logout needs
  `endSessionEndpoint` (taken from discovery) and a `postLogoutRedirectURI` registered on the app.
- **`onAPIError.errorURL` and `AUTH_ERROR_PATH`** must land on a page `proxy.ts` excludes, otherwise
  every failure restarts the flow.

The values come from the team's secrets (`auth.id`, `auth.secret`, `auth.issuer` in local-dev.md's
`secrets.yml`), each one a `required()` and a placeholder on the Dockerfile's `next build` line.

### No sign-in page: a route that starts the flow

With a single IdP, the usual `/sign-in` page is a button that does nothing but redirect. A route
handler can start the flow directly. `signInSocial` keeps the OAuth state and the PKCE verifier in
cookies, so its `Set-Cookie` headers must be copied onto the redirect:

```ts
// src/app/api/auth/login/route.ts — a specific route wins over the [...all] catch-all
import { type NextRequest, NextResponse } from "next/server";
import { AUTH_ERROR_PATH, PROVIDER_ID } from "@/lib/auth/provider";
import { auth } from "@/lib/auth/server";

export async function GET(request: NextRequest) {
  const callbackURL = safePath(request.nextUrl.searchParams.get("callbackURL"));
  const { headers, response } = await auth.api.signInSocial({
    body: { provider: PROVIDER_ID, callbackURL, errorCallbackURL: AUTH_ERROR_PATH },
    headers: request.headers,
    returnHeaders: true,
  });
  if (!response.url) return NextResponse.redirect(new URL(AUTH_ERROR_PATH, request.url));

  const redirect = NextResponse.redirect(response.url);
  for (const cookie of headers.getSetCookie()) redirect.headers.append("set-cookie", cookie);
  return redirect;
}

// Same-origin paths only: anything else would turn the sign-in into an open redirect.
function safePath(value: string | null): string {
  return value?.startsWith("/") && !value.startsWith("//") && !value.startsWith("/\\") ? value : "/";
}
```

`proxy.ts` sends a request without a session cookie to `/api/auth/login?callbackURL=<path>`; the
matcher excludes `api/auth` and the error page. The `safePath` guard is not optional: `callbackURL`
comes from the query string. The Hydra section below shows the other way to do it — starting the
flow from `proxy.ts` itself — and everything there about `trustedProxyHeaders`, `sub` and the
redirect loop applies here too.

## Two ways to persist a session: DB-backed (default) vs stateless

**DB-backed is the default and the right starting point.** Every session is
a row in the `session` table (see prisma.md); the cookie only holds an
opaque token referencing that row. `auth.api.getSession()` looks it up (an
optional `session.cookieCache` config can serve reads from a short-lived
signed cookie for a few seconds to cut down on DB round-trips, without
changing the DB as the source of truth). This fits Shape A/C naturally (the
app owns its own database, see services-and-errors.md) — there's already a
connection open for everything else.

**Stateless — the `jwt` plugin — is the exception, for one specific case:**
Shape B, a thin BFF in front of an external API that needs to verify the
caller *itself*, without calling back into the Next.js app's session store.
Add the plugin and the app additionally mints a self-contained, signed JWT
(with a JWKS endpoint the upstream API can verify against) alongside the
normal DB session:

```ts
import { jwt } from "better-auth/plugins";

export const auth = betterAuth({
  // ...database, provider config from above...
  plugins: [jwt() /* , genericOAuth(...) / socialProviders */],
});
```

```ts
// src/lib/api.ts — Shape B, forwarding a self-verifiable token instead of the IdP's own access token
const { token } = await auth.api.getToken({ headers: await headers() });
config.headers.Authorization = `Bearer ${token}`;
```

This is a *different* token than the one the Token refresh section below
forwards — that one is the original IdP's OAuth access token, useful when
the upstream API already trusts that IdP directly. The `jwt` plugin's token
is minted *by this app*, useful when the upstream API is meant to trust the
Next.js app itself as an identity boundary instead (or can't maintain a live
lookup against it). Default to DB-backed sessions and the IdP's own access
token; reach for `jwt()` only once a specific upstream consumer actually
needs a self-verifiable token instead of a session/token lookup.

**A Shape B app with no database at all** (an admin console on Ory Hydra) is a third option and
is neither of these: better-auth with no `database`, session and tokens in encrypted cookies —
see "Ory Hydra, stateless (no database)" below.

## Ory Hydra, stateless (no database) — the worked case

A Shape B app (a BFF/admin console with no database of its own) signing in against Ory Hydra
doesn't need the DB-backed default above, nor the `jwt()` plugin (that one mints a *new* token for
an upstream API). It needs better-auth with **no `database` at all**: the session lives in an
encrypted (JWE) cookie and the OAuth tokens in an encrypted account cookie. This is what
`xdew-cloud/console` and `zoug-night/admin` run. Get every point below right, the failure mode of
each one is the same symptom — an endless `app → Hydra → app → Hydra` redirect loop.

```ts
// src/lib/auth/provider.ts — shared by the server instance and the proxy
// The redirect URI registered on the Hydra client is <origin>/api/auth/callback/<PROVIDER_ID>.
export const PROVIDER_ID = "hydra";
```

```ts
// src/lib/auth/server.ts
export const auth = betterAuth({
  secret: required("AUTH_SECRET"),        // 32+ chars: openssl rand -base64 32
  baseURL: required("AUTH_URL"),
  // Behind an ingress the protocol is mis-detected without this (http instead of https), the
  // session cookie is mis-set or dropped, and the proxy restarts sign-in on every request.
  advanced: { trustedProxyHeaders: true },
  session: {
    expiresIn: SESSION_SECONDS,
    cookieCache: { enabled: true, strategy: "jwe", maxAge: SESSION_SECONDS, refreshCache: true },
  },
  account: { storeAccountCookie: true, storeStateStrategy: "cookie" }, // both default on with no database
  user: {
    additionalFields: {
      // `input: true` is what lets mapProfileToUser fill them: provider-profile fields declared
      // `input: false` are dropped (and a `required` one then fails with MISSING_FIELD).
      username: { type: "string", required: false, input: true },
      role: { type: "string", required: false, input: true },
    },
  },
  disabledPaths: ["/update-user"],        // nothing else may write those fields
  plugins: [
    genericOAuth({
      config: [{
        providerId: PROVIDER_ID,
        clientId: required("CLIENT_ID"),
        clientSecret: required("CLIENT_SECRET"),
        discoveryUrl: `${required("IDENTITY_ISSUER")}/.well-known/openid-configuration`,
        scopes: ["openid", "email", "profile", "offline_access"], // offline_access = refresh token
        pkce: true,
        authentication: "basic",           // Hydra client registered with client_secret_basic
        mapProfileToUser: (profile) => ({
          username: String(profile.preferred_username || profile.sub),
          // custom claims (role, groups) from the token hook; see getUserInfo below if userinfo lacks them
        }),
      }],
    }),
    nextCookies(),                         // last: lets server actions set cookies
  ],
});
```

Things that are easy to get wrong:

- **`sub` is the account key.** For an OIDC provider better-auth keys the account on
  `profile.sub`. A custom `getUserInfo` that returns only `id` fails the callback with
  `OAUTH_ACCOUNT_SUBJECT_INVALID` (visible in the server logs, nowhere in the browser) — return
  `sub` (and `id`). Only write a `getUserInfo` when a claim you need (e.g. a `role` injected by
  Hydra's token hook) is in the ID token but not echoed by the userinfo endpoint; otherwise let
  the discovered userinfo + `mapProfileToUser` do the work. Decode the ID token with
  `Buffer.from(token.split(".")[1], "base64url")`.
- **The provider id is part of the redirect URI.** `PROVIDER_ID = "hydra"` ->
  `/api/auth/callback/hydra`. In better-auth 1.7 the generic provider is served through the core
  `signIn.social` + `callback/:id` endpoints: there is **no `genericOAuthClient`** and no
  `/oauth2/callback/` path. Match the id to the URI already registered in the Hydra
  `OAuth2Client` (the Helm chart's `redirectUris`) instead of re-registering the client.
- **`getSessionCookie` only proves a cookie exists.** The authoritative check stays `session()` /
  `getUser()`.
- **Same version of the env contract as before:** `AUTH_SECRET`, `AUTH_URL`, `CLIENT_ID`,
  `CLIENT_SECRET`, `IDENTITY_ISSUER` (better-auth also reads `BETTER_AUTH_SECRET`), plus `API_URL`.

### No sign-in page: the proxy starts the flow

A request without a session cookie is sent straight to Hydra from `proxy.ts`; there is no
`/sign-in` route and no client-side button. `signInSocial` stores the OAuth state and PKCE
verifier in cookies, so those `Set-Cookie` headers must be copied onto the redirect response:

```ts
// src/lib/auth/proxy.ts
export async function redirectToSignIn(req: NextRequest) {
  const { headers, response } = await auth.api.signInSocial({
    body: {
      provider: PROVIDER_ID,
      callbackURL: req.nextUrl.pathname + req.nextUrl.search,
      // A failed callback must land on a page the proxy excludes, never back on the requested
      // URL — otherwise every failure restarts the flow forever.
      errorCallbackURL: "/api/auth/error",
    },
    headers: req.headers,
    returnHeaders: true,
  });
  if (!response.url) throw new InternalServerError("Identity provider returned no authorization URL");

  const redirect = NextResponse.redirect(response.url);
  for (const setCookie of headers.getSetCookie()) redirect.headers.append("set-cookie", setCookie);
  return redirect;
}
```

The matcher excludes `api/auth`, `error`, `_next/*`, `favicon.ico` and files with an extension.

### Refresh tokens in the proxy, not at render time

Hydra **rotates refresh tokens on use**, and a Server Component cannot write cookies — so a
refresh done while rendering is lost and the next request replays a consumed token (signed out).
Refresh in `proxy.ts`, where the rotated tokens can be written back, and share the in-flight
result across concurrent requests (pages, prefetches and actions arrive together):

```ts
// src/lib/auth/proxy.ts
const refreshCache = new Map<string, { promise: Promise<Headers>; expiresAt: number }>(); // 30s TTL

export function refreshSession(requestHeaders: Headers): Promise<Headers> {
  const key = requestHeaders.get("cookie") ?? "";
  // ...prune expired entries, return the cached promise if present...
  const promise = auth.api
    .getAccessToken({ body: { useAccountCookie: true }, headers: requestHeaders, returnHeaders: true })
    .then(({ headers }) => headers); // refreshes only if expired; carries the rotated Set-Cookie
  // ...cache it...
  return promise;
}

// withCookies(requestHeaders, setCookies): merge those Set-Cookie values into the *request's*
// Cookie header so the Server Components rendered in this same request already see the new tokens.
```

```ts
// src/proxy.ts
export default async function proxy(req: NextRequest) {
  if (!getSessionCookie(req)) return redirectToSignIn(req);
  try {
    const setCookies = (await refreshSession(req.headers)).getSetCookie();
    const response = NextResponse.next({ request: { headers: withCookies(req.headers, setCookies) } });
    for (const c of setCookies) response.headers.append("set-cookie", c);
    return response;
  } catch (error) {
    console.error("Error refreshing access token:", error); // refresh token rejected: dead session
    return redirectToSignIn(req);
  }
}
```

Services then just read the (already fresh) token — `getAccessToken({ body: { useAccountCookie: true } })`
in the axios request interceptor of `lib/api.ts`; `useAccountCookie` is the DB-less form of the
call (`providerId` is the DB form).

### Client and sign-out

`lib/auth/client.ts` is just `createAuthClient()` (no plugins). Sign out with
`signOut({ fetchOptions: { onSuccess: () => router.refresh() } })`. That clears the *app* session
only: while Hydra's own SSO session lives, the next request signs the user straight back in. A real
logout needs Hydra's end-session endpoint and a `post_logout_redirect_uri` registered on the
client — only add that if the product needs it.

### The build must not need the secret

`betterAuth()` runs at import time and refuses to load in production without `AUTH_SECRET`, which
breaks `next build` in CI (the Docker builder stage has no env). Pass placeholders inline on the
`RUN ... next build` command — see tooling.md, "Env that `next build` needs at import time".

### Debugging the loop

`kubectl logs deploy/<app>` — the browser only ever sees redirects. Look for
`OAUTH_ACCOUNT_SUBJECT_INVALID` (missing `sub`), `MISSING_FIELD` (additional field needs
`input: true`), `state_mismatch`/dropped cookies (missing `trustedProxyHeaders`, wrong
`AUTH_URL`), and `invalid_grant` (a rotated refresh token replayed: refresh outside the proxy).

## `src/proxy.ts` — the route guard

Next.js 16 renamed `middleware.ts` to `proxy.ts` (same mechanism, see
SKILL.md). better-auth's own guidance is explicit here: don't call
`auth.api.getSession()` (a real DB round-trip) from the proxy — check for
the session **cookie's presence** only, and do the real, authoritative
session check in the layout/page/service that actually needs it. This is an
optimistic check, not authorization — see permissions.md for where the
real permission check happens.

```ts
import { type NextRequest, NextResponse } from "next/server";
import { getSessionCookie } from "better-auth/cookies";
import { PROTECTED_ROUTES } from "./constants/routes";

export default async function proxy(req: NextRequest) {
  const { pathname } = req.nextUrl;

  if (Object.values(PROTECTED_ROUTES).some(({ path }) => path.test(pathname))) {
    const sessionCookie = getSessionCookie(req);
    if (!sessionCookie) {
      const signInUrl = new URL("/sign-in", req.url);
      signInUrl.searchParams.set("callbackUrl", req.url);
      return NextResponse.redirect(signInUrl);
    }
  }

  return NextResponse.next();
}

export const config = {
  matcher: ["/((?!api/auth|sign-in|error|_next/static|_next/image|favicon.ico|.*\\.[a-zA-Z]+$).*)"],
};
```

`getSessionCookie` only parses/validates the cookie shape — no database
call, no need to lazily import the full `auth` instance the way a
JWT-callback-based library needed to keep its server bundle out of the edge
runtime; there's nothing heavy to keep out here in the first place.
`PROTECTED_ROUTES` is the same `Record<string, { path: RegExp }>` shape as
before (`src/constants/routes.ts`, or `permissions.ts` in apps using the
`admin`-plugin access control — see permissions.md). Always exclude
`api/auth`, the sign-in page itself, `error`, static assets, and
files-with-extensions from the matcher. Apps with public SEO pages also exclude the
extension-less `opengraph-image`/`twitter-image` routes — see seo.md.

Unlike a library with its own hosted sign-in page, better-auth expects the
app to own a real `/sign-in` route — a normal page that calls
`authClient.signIn.social(...)` (built-in provider) or
`authClient.signIn.oauth2(...)` (`genericOAuth`) client-side, typically from
a single "Sign in" button, reading the `callbackUrl` search param proxy.ts
set and passing it through as `callbackURL`.

## `src/app/api/auth/[...all]/route.ts` — the one route handler

```ts
import { toNextJsHandler } from "better-auth/next-js";
import { auth } from "@/lib/auth/server";

export const { GET, POST } = toNextJsHandler(auth);
```

One catch-all route, not two — better-auth doesn't need a separate
sign-in-redirect entrypoint the way a library whose `signIn()` call has to
happen server-side does; `authClient.signIn.*` triggers the OAuth redirect
directly from the client, so there's nothing for a second route to do.

## `src/lib/auth/client.ts` — the client instance, no provider needed

```ts
"use client";
import { createAuthClient } from "better-auth/react";
import { genericOAuthClient } from "better-auth/client/plugins"; // only if using the genericOAuth plugin above

export const authClient = createAuthClient({
  plugins: [genericOAuthClient()],
});

export const { useSession, signIn, signOut } = authClient;
```

`useSession()` works directly, anywhere in the client tree — better-auth's
React client is backed by an external store, not React context, so there's
**no `<SessionProvider>` wrapper needed** in the root layout (unlike the
`providers/session.tsx` pattern an older, context-based auth library
needed — see file-organization.md's `providers/` section, which still
applies to other providers, just not this one). The root layout stays a
plain server component with no auth-specific wiring at all beyond whatever
`getUser()`-based data it fetches itself.

## Token refresh

better-auth stores the OAuth `accessToken`/`refreshToken`/
`accessTokenExpiresAt` on the `account` row itself and refreshes
automatically when asked for a current token — no hand-rolled refresh-cache
/ deduplication logic to write (a real problem with a JWT-in-cookie
approach, effectively solved here by the fact that the token lives
server-side in a table, not duplicated into a client-readable cookie).
Where a service needs a live access token to call an upstream API (the
Shape B BFF pattern in services-and-errors.md), fetch it through the API
instead of reading a stale value off the session:

```ts
// src/lib/api.ts
export const api = axios.create({ baseURL: `${required("API_URL")}/v1` });
api.interceptors.request.use(async (config) => {
  const { headers } = await import("next/headers");
  const { accessToken } = await auth.api.getAccessToken({
    body: { providerId: "identity-provider" },
    headers: await headers(),
  });
  if (accessToken) config.headers.Authorization = `Bearer ${accessToken}`;
  return config;
});
```

`getAccessToken` refreshes under the hood if the stored token is expired,
using the stored refresh token — this replaces the old
`REFRESH_BUFFER_SECONDS` + in-flight-promise-cache pattern entirely; that
complexity existed specifically to avoid racing a rotating refresh token
across concurrent requests in a JWT-cookie design, and doesn't apply once
the token lives in one place (the database) instead of being re-derived
per-request from a signed cookie.

## Optional plugin: two-factor / TOTP

```ts
import { twoFactor } from "better-auth/plugins";

export const auth = betterAuth({
  // ...database, provider config from above...
  plugins: [
    twoFactor({ issuer: "MyApp" }), // shown inside the user's authenticator app
    // ...genericOAuth/socialProviders, admin, etc.
  ],
});
```

```ts
// src/lib/auth/client.ts
import { twoFactorClient } from "better-auth/client/plugins";

export const authClient = createAuthClient({
  plugins: [twoFactorClient(), genericOAuthClient()],
});
```

Enrollment is a client-side flow: `authClient.twoFactor.enable({ password })`
returns a `totpURI` (render it as a QR code for the user's authenticator
app) and a set of backup codes; `authClient.twoFactor.verifyTotp({ code })`
confirms the six-digit code at sign-in once it's enabled.

**The friction point for this stack's default setup:** `enable`/`disable`
require the account's *password* to re-confirm a security-sensitive
change — but every example in this file so far is OAuth-only
(`genericOAuth`/`socialProviders`), with no password ever set. An
OAuth-only account can't call `enable` as written above. Two ways around
it, pick based on what the app actually needs:

- Also turn on better-auth's `emailAndPassword` provider, even if OAuth
  stays the *primary* sign-in method — the password exists only to gate
  2FA enrollment/changes, not for everyday sign-in.
- Skip TOTP entirely and rely on the IdP's own MFA instead. If every user
  signs in through an enterprise IdP (Entra ID, Ory Hydra, etc.), that IdP
  is very likely already enforcing its own second factor upstream, and an
  app-level TOTP layer is redundant rather than additive. TOTP earns its
  place specifically when the app supports (or primarily uses) email/password
  sign-in and needs a second factor independent of any IdP.

Check the installed version's docs for the current re-authentication story
before assuming the password requirement above is exactly as described —
this is precisely the kind of plugin-surface detail the epistemic note at
the top of this file is about.

## Optional plugin: Stripe billing (`@better-auth/stripe`)

Billing (subscriptions, per-user or per-organization, plan-limit
enforcement, webhooks) has its own file — see stripe.md. It follows the
same plugin-pairing shape as everything above (a server plugin plus its
client counterpart, routes mounted under the same `api/auth/[...all]`
catch-all, its own CLI-generated tables) — stripe.md picks up from here.

## Never bypass `getUser()`/`session()`

Every service function that needs the current user calls `getUser()` (or
`session()` directly if `null` is a meaningful, handled outcome rather than
an error) — don't read cookies/headers manually or call
`auth.api.getSession()` ad hoc to reconstruct identity in a one-off place.

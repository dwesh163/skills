# Services and errors

Three valid shapes for the service layer, depending on what the app owns and
how much it needs to protect. Check which one an existing repo already uses
before adding a new service — don't mix them within one codebase.

## Naming entities: `IDENTITIES`, not hardcoded strings

A domain's identifier string (`"orders"`, `"products"`, `"users"`) gets
retyped at a lot of call sites — `ForbiddenError(...)`, `NotFoundError(...)`,
`EntityNotFound`'s `entity` prop, an `error.tsx` wrapper's `entity` prop,
`RESOURCES`'s own keys. Define it once, reference it everywhere else — never
a bare string literal at any of those call sites:

```ts
// src/constants/resources.ts
import { type LucideIcon, Package, ShoppingBag, UserX } from "lucide-react";

export const IDENTITIES = {
  PRODUCTS: "products",
  ORDERS: "orders",
  USERS: "users",
} as const;

export type Entity = (typeof IDENTITIES)[keyof typeof IDENTITIES];

export const RESOURCES: Record<Entity, { icon: LucideIcon }> = {
  [IDENTITIES.PRODUCTS]: { icon: Package },
  [IDENTITIES.ORDERS]: { icon: ShoppingBag },
  [IDENTITIES.USERS]: { icon: UserX },
};
```

```tsx
throw new ForbiddenError(IDENTITIES.PRODUCTS);   // not ForbiddenError("products")
<EntityNotFound entity={IDENTITIES.ORDERS} backHref="/orders" />  // not entity="orders"
```

`Entity` is *derived* from `IDENTITIES`, not declared as a separate union by
hand — extend `IDENTITIES` (and `RESOURCES` alongside it) when a new entity
gets a detail-page not-found state or a permission check, and every typed
consumer of `Entity` picks it up automatically.

## Shape A — owns the database, sanitized responses

The most defensive shape: worth it once a service's errors might otherwise
leak internal details (Prisma/driver error text, query shape) to the client,
or once there's more than a couple of domains and consistency across all of
them starts to matter.

Every file in `src/services/` starts with `"use server";`. Every exported
function returns `ServiceResponse<T>` and never throws to the caller.

```ts
// src/types/response.ts
export type ServiceResponse<T> = { data: T | null; error: string | null };

export async function toResponse<T>(fn: () => Promise<T>, fallback: string): Promise<ServiceResponse<T>> {
  try {
    return { data: await fn(), error: null };
  } catch (error) {
    const expected = error instanceof ExpectedError;
    console.error(fallback, error);
    if (!expected) await log.error("service", error instanceof Error ? error.message : fallback, { fallback });
    return { data: null, error: expected ? error.code : "unknown" };
  }
}
```

```ts
// src/services/products.ts
"use server";
export async function update(id: number, data: Partial<Product>): Promise<ServiceResponse<Product>> {
  return toResponse(async () => {
    const user = await getUser();
    const { success } = await auth.api.userHasPermission({ body: { userId: user.id, permissions: { product: ["update"] } } });
    if (!success) throw new ForbiddenError(IDENTITIES.PRODUCTS);
    return prisma.products.update({ where: { id }, data });
  }, "Failed to update product");
}
```

- Check permissions first (see permissions.md), then do the Prisma call.
- CRUD verb names: `list`, `get`, `create`, `update`, `remove`, `reorder` —
  not `getAll`, `delete`, `fetch`. Stay consistent with sibling services.
- **Why `toResponse` sanitizes:** it only lets an `ExpectedError` subclass's
  `.code` reach the client — one of `notFound`, `forbidden`, `unauthorized`,
  `badRequest`, `internal`, or a `RuleError`'s `rules.<rule>`. Anything else —
  a bare `Error`, a `PrismaClientKnownRequestError`, a driver error —
  collapses to the generic `"unknown"` code, because Prisma/driver messages
  can leak raw query/column details to the browser. The `fallback` string is
  still logged server-side, so debugging isn't lost, it's just not shown to
  the user. **Never** catch a Prisma error yourself and return
  `error.message` directly, and never add a new error class without
  extending `ExpectedError` — an unrecognized class is treated as unexpected
  and collapses to `"unknown"` too.
- On the client: never `toast.error(error)` with a service's raw error
  string. `error` is exactly one of the codes above — key straight into
  `errors.*` with it instead of just checking whether it's non-null (see
  translations.md).
- **Ordering with not-yet-persisted rows:** a client-side "new" item placed
  in a list before it's saved (temp `id: Date.now()`) doesn't have a real
  database id yet. Any batch op that assumes every row exists (`reorder`,
  bulk update-by-id) must run *after* the per-item create step that gives
  each row its real id — using the id that call returned, not the original
  array. Calling `reorder`/`update` with a placeholder id fails against
  Prisma in a way that's easy to miss if you don't check every step's
  `{ error }`.

## Shape B — thin BFF over an external REST API

When the Next.js app doesn't own the database and instead proxies a separate
backend service, services skip `ServiceResponse`/`toResponse` and just await
an authenticated HTTP client, letting errors throw and bubble to the nearest
error boundary (see below) or a caller's own try/catch:

```ts
// src/lib/api.ts
export const api = axios.create({ baseURL: `${process.env.API_URL}/v1` });
api.interceptors.request.use(async (config) => {
  const session = await auth();
  if (session?.user?.accessToken) config.headers.Authorization = `Bearer ${session.user.accessToken}`;
  return config;
});
```

```ts
// src/services/organisations.ts
export const organisations = {
  list: () => api.get<Organisation[]>("/organisations").then((r) => r.data),
  get: (id: string) => api.get<OrganisationDetail>(`/organisations/${id}`).then((r) => r.data),
  async create(input: CreateOrganisationInput) {
    "use server";
    await api.post("/organisations", input);
    revalidatePath("/organisations");
  },
};
```

One namespace object per domain (`export const organisations = {...}`), not
a flat `listOrganisations`/`createOrganisation`/`updateOrganisation` export
per verb — the call site reads as `organisations.list()`,
`organisations.create(input)`, which is both shorter and self-documenting
about which domain it belongs to without a `Service` suffix. Keep a method
one-line (arrow, direct return) whenever the body is a single expression;
reach for the block-bodied `async name(...) {}` shorthand only once there's
real logic in the body (a `"use server"` directive, more than one
statement, a not-found check — see Shape C below).

**Name the namespace after the domain's plural form** (`organisations`,
`orders`, `products`), not the singular. This is what makes
`organisations.get(id)` collision-free at the call site: the plural is what
you import and call methods on, which leaves the singular free for the
record you actually fetched —

```ts
const organisation = await organisations.get(id);
```

Naming the namespace singular (`organisation.get(id)`) collides with exactly
the local variable name the call site wants next; pluralizing the namespace
is the fix, not renaming the local variable to something awkward
(`organisationRecord`, `org`) just to dodge the shadow.

**`list()` reintroduces the same problem one level up** — its result is
naturally named the same plural as the namespace itself:

```ts
const organisations = await organisations.list(); // ❌ don't do this
```

This isn't just ugly, it's a real bug: `const` shadows the imported
`organisations` binding for the rest of that scope, and thanks to the
temporal dead zone, the right-hand side's `organisations.list()` resolves to
the *local, not-yet-initialized* binding, not the import — this throws
`ReferenceError: Cannot access 'organisations' before initialization` at
runtime, it doesn't just read badly. Fix it by aliasing the import in the
specific files that call `.list()` and want the plural local name:

```ts
import { organisations as organisationsApi } from "@/services/organisations";

export default async function OrganisationsPage() {
  const organisations = await organisationsApi.list();
  // ...
}
```

Only files that need *both* the namespace and a same-named local variable
need this alias — a detail page calling `.get(id)` never needs it (the
singular/plural split already keeps those collision-free), and a page that
passes the list straight through without binding it to a name doesn't
either:

```tsx
<Table rows={await organisations.list()} />
```

Alias at the import, not by renaming the service's export or changing what
`.list()` returns — the service module itself stays exactly
`export const organisations = { list, get, create, ... }` everywhere; only
the rare call site that needs the plural name twice adds one word to its
import line.

Mutations call `revalidatePath`/`revalidateTag` after the write instead of
returning updated data for the caller to splice in — the next render just
refetches. Read-only methods (`list`, `get`) don't need `"use server"` if
the whole file isn't already marked, since they're plain data fetches, not
mutations; only methods that change server state need the directive
explicitly (as their first statement) when the file itself isn't
`"use server"` top-to-bottom.

In Shape B/C a service that throws is fine for a page's *own* data (it reaches
`error.tsx`), but any read that is only *secondary* to the page — a side panel,
a list in the layout — must go through `load` (see "`load`" below) so its
failure is shown, not swallowed.

Pick Shape A when the app is the system of record; pick Shape B when it's a
dashboard/admin console in front of another service's API.

## The `ExpectedError` hierarchy (`src/constants/errors.ts`)

```ts
export abstract class ExpectedError extends Error {
  abstract readonly code: string;
}

export class NotFoundError extends ExpectedError {
  readonly code = "notFound";
  constructor(readonly entity?: Entity) { super(entity ? `${entity} not found` : "Not found"); this.name = "NotFoundError"; }
}
export class ForbiddenError extends ExpectedError {
  readonly code = "forbidden";
  constructor(readonly entity?: Entity) { super(entity ? `${entity} access denied` : "Forbidden"); this.name = "ForbiddenError"; }
}
export class UnauthorizedError extends ExpectedError {
  readonly code = "unauthorized";
  constructor() { super("User not authenticated"); this.name = "UnauthorizedError"; }
}
// A business-rule rejection with a specific translated message. `.code` reproduces the
// "rules.<key>" wire format so the client can key a translation off it directly.
export class RuleError extends ExpectedError {
  readonly code: string;
  constructor(readonly rule: string) { super(rule); this.name = "RuleError"; this.code = `rules.${rule}`; }
}
// Malformed/invalid input caught before it reaches Prisma (upload validation, etc).
export class BadRequestError extends ExpectedError {
  readonly code = "badRequest";
  constructor(message: string) { super(message); this.name = "BadRequestError"; }
}
// A recognized internal/infra failure (downstream integration error, missing seed row) — an
// ExpectedError like the others, so the client gets the stable "internal" code instead of
// falling into the same "unknown" bucket as a truly unanticipated exception. `message` is
// still never sent to the client (see toResponse above) — only `.code` crosses that boundary.
export class InternalServerError extends ExpectedError {
  readonly code = "internal";
  constructor(message: string) { super(message); this.name = "InternalServerError"; }
}
```

Five codes in total reach the client this way: `notFound`, `forbidden`,
`unauthorized`, `badRequest`, `internal` — plus the sanitizer's own
`"unknown"` fallback for anything that isn't one of these (and `RuleError`'s
dynamic `rules.<rule>`, which is a separate, per-rule namespace rather than a
fixed code). translations.md's `errors.*` block carries exactly one key per
fixed code, `unknown` included — never fewer, never an extra one that no
class or fallback actually produces.

`entity` is typed as `Entity` (from `IDENTITIES`, see above), not a bare
`string` — passing a typo'd or ad hoc string literal here is a compile
error, not just a convention to remember.

Throw the specific class at the point of failure — never a bare
`new Error("...")` sentinel string. A record that should exist but doesn't:
`NotFoundError(entity)`. A permission check that fails:
`ForbiddenError(entity)`. A recognized-but-internal failure you want to name
instead of an anonymous `Error`: `InternalServerError(message)` — the client
gets the stable `internal` code instead of the generic `unknown` fallback,
while `message` itself still never leaves the server. A truly unanticipated
exception (Prisma crashing) doesn't need catching and rethrowing as
anything — `toResponse`'s catch-all already collapses it to `unknown`.

## Shape C — owns the data, throws directly (small apps)

A small app — few domains, one team, no untrusted client ever seeing a raw
error — doesn't need the `ServiceResponse`/`toResponse` wrapper even though
it owns its data directly (Prisma, or something else entirely — e.g. a set
of Kubernetes/Helm releases standing in for a database, see prisma.md). The
error classes can skip the abstract `ExpectedError` base too — each one just
declares its own `readonly code` on top of plain `Error`:

```ts
// src/constants/errors.ts — no shared ExpectedError base class needed at this scale
export class NotFoundError extends Error {
  readonly code = "notFound";
  readonly resource: string;
  constructor(resource = "Resource") { super(`${resource} not found`); this.name = "NotFoundError"; this.resource = resource; }
}
```

Same namespace-object export as Shape B above (one object per domain, named
after its **plural** form so it doesn't collide with a locally fetched
singular record — `stages.get(id)` called into `const stage = ...`), and
methods `throw` directly instead of returning `ServiceResponse` — the caller
(a Server Component awaiting the call, or an `error.tsx` boundary) handles
it:

```ts
// src/services/stages.ts
export const stages = {
  list: () => prisma.stage.findMany({ orderBy: { start: "asc" } }),
  async get(id: string) {
    const { workspaceId } = await user.read();
    const stage = await prisma.stage.findFirst({ where: { id, workspaceId } });
    if (!stage) throw new NotFoundError(IDENTITIES.STAGES);
    return stage;
  },
  create: (data: StageInput) => prisma.stage.create({ data }),
};
```

Any of the three shapes is a fine way to start a new app — but **never** a
bare `Error` for an expected-and-nameable failure in any of them; always one
of the named classes above so the failure is recognizable at the boundary
that needs to react to it. If a small app (Shape C) grows into needing
sanitized client-facing errors later, migrating to Shape A is mostly
mechanical: wrap each service body in `toResponse`, make the error classes
extend a shared `ExpectedError`.

## The not-found / error trio — three distinct situations

Don't reach for the wrong one.

**1. A route doesn't exist at all** — `src/app/not-found.tsx`, Next's root
catch-all for unmatched URLs. You don't render this directly; Next calls it.

**2. A specific record doesn't exist** (product/order/user not found) —
`src/components/not-found.tsx` exports `EntityNotFound`:

```tsx
// Shape A: the ServiceResponse carries the miss
if (!order) return <EntityNotFound entity={IDENTITIES.ORDERS} backHref="/orders" />;

// Shape B/C: the service throws NotFoundError; `load.optional` turns only that into null
const order = await load.optional(orders.get(id));
if (!order) return <EntityNotFound entity={IDENTITIES.ORDERS} backHref="/orders" />;
```

Render it directly once a fetch returns nothing — don't throw for this case.
`backHref` is the entity's list route (the "back" button label is the
entity's plural title). Use `load.optional` (see the `load` section below),
never `.catch(() => null)`, so a real failure isn't reported as "not found".
`entity` is an `IDENTITIES` value (see above), never a bare string literal.
Works from both a client component and an `async` server component
(`EntityNotFound` is itself `"use client"`, a server component can still
render it as a child).

**3. An unexpected error during render** (React error boundary) —
`src/app/error.tsx` exports the shared error-boundary UI (`error`, `reset`,
`entity` props). Per-route `error.tsx` files are thin wrappers:

```tsx
"use client";
import AppError from "@/app/error";

export default function OrderError({ error, reset }: { error: Error & { digest?: string }; reset: () => void }) {
  return <AppError error={error} reset={reset} fullScreen={false} entity={IDENTITIES.ORDERS} />;
}
```

Add one next to any route segment where a thrown exception should show a
scoped error instead of bubbling to the root boundary. Note: a route
`error.tsx` boundary only ever gets `message`/`digest` off the thrown error
in production (Next.js strips other properties crossing that boundary) — it
can't read a typed `ExpectedError`'s `.code`/`.entity`. That's fine, the
typed-error path targets `ServiceResponse`, not thrown-and-caught render
exceptions; this boundary stays generic on purpose.

**Inline data-loading failure** (not a boundary, not a missing record) —
`src/components/error.tsx` exports `ErrorCard`, a small card for "this
section's data failed to load" inside an otherwise-working page. Use this
instead of the full `AppError` boundary when the rest of the page is still
usable and only one panel failed. Feed it with `load()` (next section), not
a `.catch()`.

## `load` — never swallow an error with `.catch(() => [])`

`.catch(() => [])` / `.catch(() => null)` on a read makes a failure look
identical to "there is nothing here": the user sees an empty list and nobody
knows something broke. Don't write it. Every read either **throws** (to the
nearest `error.tsx`) or goes through `src/lib/load.ts`, which makes the
failure a value the page can render. `load` is one callable with three
helpers attached, so there is a single import:

```ts
// src/lib/load.ts
import { ExpectedError, NotFoundError } from "@/constants/errors";

export type Loaded<T> = { data: T; error: string | null };

// The stable code the client keys into `errors.*`; anything unexpected is "unknown".
function code(error: unknown): string {
  return error instanceof ExpectedError ? error.code : "unknown";
}

// Non-critical read: the page still renders without the data. A falsy `promise`
// (`null`/`false`/`undefined`) is skipped and resolves to `fallback`.
async function soft<T>(promise: Promise<T> | null | false | undefined, fallback: T): Promise<Loaded<T>> {
  if (!promise) return { data: fallback, error: null };
  try {
    return { data: await promise, error: null };
  } catch (error) {
    console.error(error); // logged server-side; only the code reaches the client
    return { data: fallback, error: code(error) };
  }
}

// A missing record is `null`; every other failure still throws to the boundary.
async function optional<T>(promise: Promise<T>): Promise<T | null> {
  try {
    return await promise;
  } catch (error) {
    if (error instanceof NotFoundError) return null;
    throw error;
  }
}

// First error code among several results, so one ErrorCard covers a batch.
function failure(results: { error: string | null }[]): string | null {
  return results.find((r) => r.error)?.error ?? null;
}

export const load = Object.assign(soft, { optional, code, failure });
```

Pick the helper by what the read means for the page:

| The read is… | Use | On failure |
|---|---|---|
| the page's own record (detail page) | `load.optional(x.get(id))` → `<EntityNotFound />` on `null` | non-404 errors reach `error.tsx` |
| a secondary panel (a list, a side card) | `load(x.list(), [])` → `{ data, error }` | `<ErrorCard error={error} />` next to/instead of the panel |
| several secondary reads | `load.failure(results)` | one `ErrorCard` for the first failure |
| chrome in the root layout (org list, search index) | `load(...)` | a banner `ErrorCard` above `children` |

```tsx
// detail page
const order = await load.optional(orders.get(id));
if (!order) return <EntityNotFound entity={IDENTITIES.ORDERS} backHref="/orders" />;

// secondary panel
const { data: logs, error: logsError } = await load(logsApi.list(), []);
// ...
{logsError && <ErrorCard error={logsError} />}

// conditional read (e.g. only when signed in): no ternary, the falsy value is skipped
const { data: organisations, error: organisationsError } = await load(session && organisationsApi.list(), []);

// a batch
const results = await Promise.all(projects.map((p) => load(projectsApi.get(p.id), null)));
const details = results.map((r) => r.data).filter((p) => p !== null);
const error = load.failure(results);
```

`ErrorCard` is an async server component that renders `errors.<code>`
(translations.md) — it takes the *code*, never a message, for the same reason
`toResponse` never returns `error.message` (see above):

```tsx
// src/components/error.tsx
export async function ErrorCard({ error, className }: { error: string; className?: string }) {
  const translations = { errors: await getTranslations("errors") };
  return (
    <div role="alert" className={cn("flex items-center gap-2 rounded-lg border border-destructive/30 bg-destructive/10 px-3 py-2 text-sm text-destructive", className)}>
      <TriangleAlert className="size-4 shrink-0" />
      {translations.errors(error)}
    </div>
  );
}
```

A service that aggregates a non-critical sub-read carries the code on its
result instead of hiding it — e.g. `applications.get` returns
`{ ...app, pods, podsError }` from `load(pods.list(app), [])`, and the page
renders `ErrorCard` in place of the empty-pods message.

Two things that are *not* `load`'s job:

- **Fail-closed checks** (a permissions probe whose failure must read as
  "not allowed") may return a safe default, but still `console.error` the
  cause — never a bare `catch { return {} }`.
- **Client-side `catch` around a server action** only has a generic toast to
  show: in production an error thrown by a server action loses its class
  crossing the boundary, so there is no code to key. If a mutation needs a
  specific message, have it return a `ServiceResponse` (Shape A) instead of
  throwing.

The upstream API's status codes are the other half: in Shape B, map `404`/
`403`/`401` to `NotFoundError`/`ForbiddenError`/`UnauthorizedError` once, in
an axios response interceptor in `lib/api.ts`, so that `load.optional` and
`load.code` see the named classes rather than raw `AxiosError`s.

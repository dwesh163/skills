# SEO & metadata: titles, Open Graph, sitemap, robots, icons

Everything a crawler or a link-preview reads, built with Next's file-based
metadata API — no `next-seo`, no hand-written `<head>` tags. Public,
indexable pages get full metadata; everything behind `proxy.ts` gets
`noindex` and stays out of the sitemap.

```
src/
├── app/
│   ├── layout.tsx              — site-wide `metadata` (title template, OG defaults, metadataBase)
│   ├── opengraph-image.tsx     — default OG image (generated, `next/og`)
│   ├── sitemap.ts              — /sitemap.xml
│   ├── robots.ts               — /robots.txt
│   ├── manifest.ts             — /manifest.webmanifest (optional, PWA/installable)
│   ├── icon.png · apple-icon.png · favicon.ico   — picked up by filename
│   └── <route>/page.tsx        — `generateMetadata` per page
├── constants/site.ts           — name, URL, default locale, social handles
└── lib/seo.ts                  — `buildMetadata()` + JSON-LD helpers (only if pages repeat the shape)
```

## `SITE` — one source of truth

```ts
// src/constants/site.ts
export const SITE = {
  name: process.env.NEXT_PUBLIC_APP_NAME ?? "App",
  url: process.env.NEXT_PUBLIC_APP_URL ?? "http://localhost:3000",
  twitter: "@acme", // omit if none
} as const;
```

`NEXT_PUBLIC_APP_NAME` is the display name and `NEXT_PUBLIC_APP_URL` the
canonical public origin, no trailing slash (both in `.env.example`, see
tooling.md). Every absolute URL — `metadataBase`,
sitemap entries, robots' `sitemap:` line, JSON-LD — derives from it; never
hardcode a domain or the app name in a page. In Docker/CI builds both must
be present at build time, since `NEXT_PUBLIC_*` is inlined.

## Root layout — defaults and the title template

```tsx
// src/app/layout.tsx
import type { Metadata } from "next";
import { getTranslations } from "next-intl/server";
import { SITE } from "@/constants/site";

export async function generateMetadata(): Promise<Metadata> {
  const translations = { metadata: await getTranslations("metadata") };

  return {
    metadataBase: new URL(SITE.url),
    title: { default: translations.metadata("title"), template: `%s | ${SITE.name}` },
    description: translations.metadata("description"),
    applicationName: SITE.name,
    openGraph: {
      type: "website",
      siteName: SITE.name,
      title: translations.metadata("title"),
      description: translations.metadata("description"),
      url: "/",
    },
    twitter: { card: "summary_large_image", site: SITE.twitter },
    alternates: { canonical: "/" },
  };
}
```

- `metadataBase` is what lets every relative `url`/`images`/`canonical` below
  resolve to an absolute URL — without it Next warns and OG images break.
- `title.template` means pages set only their own segment (`"Pricing"` →
  `Pricing | Acme`); `default` is the home page / fallback.
- Use `generateMetadata` (not a static `metadata` object) in the root layout
  because the description is translated — same reason the layout is already
  `force-dynamic` (tooling.md). A single-language app can use a plain
  `export const metadata`.
- Child `openGraph`/`twitter` objects **replace** the parent's, they don't
  merge field by field — a page that sets `openGraph.title` must restate
  `type`/`siteName` or use the helper below.

## Per-page metadata

```tsx
// src/app/pricing/page.tsx
export async function generateMetadata(): Promise<Metadata> {
  const translations = { page: await getTranslations("pages.pricing.metadata") };
  return buildMetadata({
    title: translations.page("title"),
    description: translations.page("description"),
    path: "/pricing",
  });
}
```

```ts
// src/lib/seo.ts
import type { Metadata } from "next";
import { SITE } from "@/constants/site";

export function buildMetadata({
  title, description, path, image, noindex,
}: { title: string; description: string; path: string; image?: string; noindex?: boolean }): Metadata {
  return {
    title,
    description,
    alternates: { canonical: path },
    openGraph: { type: "website", siteName: SITE.name, title, description, url: path, ...(image && { images: [image] }) },
    twitter: { card: "summary_large_image", title, description, ...(image && { images: [image] }) },
    ...(noindex && { robots: { index: false, follow: false } }),
  };
}
```

Don't add the helper until a second page needs it — one page can inline its
metadata. Titles ≈ 50–60 chars (the template suffix counts), descriptions
≈ 120–160, one unique pair per indexable page.

Dynamic routes (`/products/[id]`) fetch through the same service layer as
the page (services-and-errors.md) and resolve a miss with `load.optional(...)` (→ `EntityNotFound`, or `notFound()` for an indexable public page) — wrap the
fetch in `React.cache` so `generateMetadata` and the page don't query twice.

## Translations

A root `metadata` block for the site defaults, and a `metadata` sub-block
inside each page's namespace (fits the `pages.*` shape in translations.md):

```json
"metadata": { "title": "Acme — Invoicing for freelancers", "description": "..." },
"pages": { "pricing": { "metadata": { "title": "Pricing", "description": "..." } } }
```

Keep `en`/`fr` in sync like any other key. Page titles stay short (the
template adds the site name); only the root `title` carries the tagline.

## Open Graph & Twitter images

Prefer generated images over static PNGs so text stays translated and
on-brand:

```tsx
// src/app/opengraph-image.tsx
import { ImageResponse } from "next/og";
import { getTranslations } from "next-intl/server";
import { SITE } from "@/constants/site";

export const size = { width: 1200, height: 630 };
export const contentType = "image/png";
export const alt = SITE.name;

export default async function Image() {
  const translations = { metadata: await getTranslations("metadata") };
  return new ImageResponse(
    (
      <div style={{ width: "100%", height: "100%", display: "flex", alignItems: "center", justifyContent: "center", fontSize: 72, background: "#0a0a0a", color: "#fafafa" }}>
        {translations.metadata("title")}
      </div>
    ),
    size,
  );
}
```

A route segment can add its own `opengraph-image.tsx` (e.g.
`app/products/[id]/opengraph-image.tsx`, receiving `params`) and it overrides
the root one for that segment — no `images` field needed in the metadata.
`ImageResponse` supports only a flexbox/CSS subset and needs fonts loaded
explicitly if not the default. A static `opengraph-image.png` (1200×630) in
the same folder works too when there's no dynamic text.

## Icons, manifest, theme color

Drop `icon.png`/`icon.svg`, `apple-icon.png` (180×180) and `favicon.ico` into
`src/app/` — Next wires the `<link>` tags by filename. Theme color goes in
`viewport`, not `metadata`:

```ts
export const viewport: Viewport = {
  themeColor: [
    { media: "(prefers-color-scheme: light)", color: "#ffffff" },
    { media: "(prefers-color-scheme: dark)", color: "#0a0a0a" },
  ],
};
```

`manifest.ts` (`MetadataRoute.Manifest`: `name`, `short_name`, `start_url`,
`display`, `icons`) only if the app is meant to be installable.

## `sitemap.ts`

```ts
// src/app/sitemap.ts
import type { MetadataRoute } from "next";
import { LEGAL_LAST_UPDATED } from "@/constants/legal";
import { SITE } from "@/constants/site";

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const staticRoutes = ["", "/pricing"].map((path) => ({
    url: `${SITE.url}${path}`,
    changeFrequency: "monthly" as const,
    priority: path === "" ? 1 : 0.8,
  }));

  const legalRoutes = [
    { path: "/legal/privacy-policy", date: LEGAL_LAST_UPDATED.privacyPolicy },
    { path: "/legal/terms-of-use", date: LEGAL_LAST_UPDATED.termsOfUse },
  ].map(({ path, date }) => ({ url: `${SITE.url}${path}`, lastModified: date, priority: 0.3 }));

  // Dynamic public entities: query through the service layer, never Prisma directly here.
  // const products = await productService.listPublic();
  // const productRoutes = products.map((p) => ({ url: `${SITE.url}/products/${p.slug}`, lastModified: p.updatedAt }));

  return [...staticRoutes, ...legalRoutes];
}
```

- List **only public, indexable, canonical** URLs — nothing behind auth,
  nothing `noindex`, no redirects. Reuse `LEGAL_LAST_UPDATED` for
  `lastModified` (legal.md) rather than `new Date()`, which would claim
  everything changed today.
- `sitemap.ts` is cached by default; add `export const revalidate = 3600`
  when it lists DB-backed entities. More than 50 000 URLs → `generateSitemaps()`.
- An app that is entirely behind login has no sitemap; skip the file.

## `robots.ts`

```ts
// src/app/robots.ts
import type { MetadataRoute } from "next";
import { SITE } from "@/constants/site";

export default function robots(): MetadataRoute.Robots {
  const isProd = process.env.NODE_ENV === "production" && process.env.ROBOTS_INDEX !== "false";
  if (!isProd) return { rules: { userAgent: "*", disallow: "/" } };

  return {
    rules: { userAgent: "*", allow: "/", disallow: ["/api/", "/sign-in", "/error"] },
    sitemap: `${SITE.url}/sitemap.xml`,
  };
}
```

Staging/preview deployments must disallow everything — an env switch
(`ROBOTS_INDEX=false`, in `.env.example`) beats remembering to edit the file.
`robots.txt` only asks crawlers not to crawl; keeping a page **out of the
index** needs `robots: { index: false }` in its metadata (below). Don't use
both on the same URL — a blocked page can't be crawled to see its `noindex`.

## Private and utility pages: `noindex`

Anything behind the `proxy.ts` guard, plus `sign-in`, `error` and
not-found, sets:

```ts
export const metadata: Metadata = { title: "Sign in", robots: { index: false, follow: false } };
```

Put it once in the layout of the authenticated route group (e.g.
`(app)/layout.tsx`) instead of on every page, and the same on
`sign-in/page.tsx`. Titles still matter there — they name the browser tab.

## Structured data (JSON-LD)

Add only where it earns a rich result (`Organization`/`WebSite` on the home
page, `Product`, `Article`, `BreadcrumbList`, `FAQPage`):

```tsx
export function JsonLd({ data }: { data: Record<string, unknown> }) {
  return (
    <script
      type="application/ld+json"
      // biome-ignore lint/security/noDangerouslySetInnerHtml: serialized JSON, `<` escaped
      dangerouslySetInnerHTML={{ __html: JSON.stringify(data).replace(/</g, "\\u003c") }}
    />
  );
}
```

Always escape `<` — user-controlled strings in the payload could otherwise
close the script tag. Build the object from the same data the page renders
so the two can't drift.

## i18n caveat

The stack detects the locale from a cookie/`Accept-Language`, not from the
URL (translations.md), so there is **one URL per page** whatever the
language: no `hreflang`, no `/fr` sitemap duplicates, and crawlers see the
default-locale version. That's fine for apps where SEO isn't the main goal.
If search visibility per language matters, the app needs locale-prefixed
routing (`/en/...`, `/fr/...`) — a structural change to ask about before
building, not something to bolt on with `alternates.languages` alone.

## Keep the routes public in `proxy.ts`

`sitemap.xml`, `robots.txt`, `manifest.webmanifest`, `favicon.ico` and icon
files already match the "files with an extension" exclusion (auth.md). The
extension-less ones don't: add `opengraph-image` and `twitter-image` to the
matcher's exclusion list, or crawlers and link-preview bots get redirected
to `/sign-in`:

```ts
matcher: ["/((?!api/auth|sign-in|error|opengraph-image|twitter-image|_next/static|_next/image|favicon.ico|.*\\.[a-zA-Z]+$).*)"],
```

Per-segment OG images (`/products/1/opengraph-image`) end with that same
segment name — if they're public, use a prefix that covers them
(`.*/opengraph-image`).

## Checklist for a new public page

1. `generateMetadata` with a unique title + description (translated).
2. Canonical path set; OG/Twitter inherit or restate `type`/`siteName`.
3. Added to `sitemap.ts` (or produced by its dynamic query).
4. Public in `proxy.ts` if it must be reachable signed-out.
5. Checked with the page's "view source" or a link-preview/OG debugger.

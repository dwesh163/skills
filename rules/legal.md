# Legal pages (privacy policy, terms of use)

Static, translated documents at `/legal/<slug>` — one shared rendering
component driven by a `doc` prop, not one hand-written page per document:

```
src/app/legal/
├── layout.tsx              — groups the routes, no content of its own
├── privacy-policy/page.tsx
└── terms-of-use/page.tsx
src/components/legal/document.tsx  — the actual rendering, shared
src/constants/legal.ts             — one "last updated" date per document
```

## The layout only groups the routes

```tsx
// src/app/legal/layout.tsx
export default function LegalLayout({ children }: { children: React.ReactNode }) {
  return <div className="flex flex-1 flex-col">{children}</div>;
}
```

That's the whole structural contract: `/legal/*` shares one `layout.tsx`
instead of each page re-implementing its own chrome. Whatever nav/header an
app wants above its legal pages (a logo, a link back to the other document)
is app-specific presentation, not part of this convention — don't copy one
app's header into another's `legal/layout.tsx` just because it lived there.

## One shared component, not one page per document

`privacy-policy/page.tsx` and `terms-of-use/page.tsx` render identically —
title, a "last updated" line, a list of `{ heading, body }` sections — so
that markup lives once, parameterized by which document to render:

```tsx
// src/components/legal/document.tsx
import { getLocale, getTranslations } from "next-intl/server";
import { LEGAL_LAST_UPDATED } from "@/constants/legal";

export async function LegalDocument({ doc }: { doc: keyof typeof LEGAL_LAST_UPDATED }) {
  const [locale, legal] = await Promise.all([getLocale(), getTranslations(`pages.legal.${doc}`)]);
  const translations = { legal };
  const updated = new Intl.DateTimeFormat(locale, { dateStyle: "long" }).format(LEGAL_LAST_UPDATED[doc]);
  const sections = translations.legal.raw("sections") as { heading: string; body: string }[];

  return (
    <div className="flex flex-1 flex-col items-center bg-background px-4 py-16">
      <div className="flex w-full max-w-2xl flex-col gap-8">
        <div className="text-center">
          <h1 className="text-2xl font-semibold text-foreground">{translations.legal("title")}</h1>
          <p className="mt-1 text-sm text-muted-foreground">{translations.legal("updated", { date: updated })}</p>
        </div>
        <div className="flex flex-col gap-6">
          {sections.map((section) => (
            <section key={section.heading}>
              <h2 className="font-semibold text-foreground">{section.heading}</h2>
              <p className="mt-1 text-muted-foreground">{section.body}</p>
            </section>
          ))}
        </div>
      </div>
    </div>
  );
}
```

```tsx
// src/app/legal/privacy-policy/page.tsx
import { LegalDocument } from "@/components/legal/document";

export default function PrivacyPolicyPage() {
  return <LegalDocument doc="privacyPolicy" />;
}
```

`terms-of-use/page.tsx` is the same shape with `doc="termsOfUse"`. Adding a
third document (cookie policy, a DPA) means one more `page.tsx` this size and
one more key on `LEGAL_LAST_UPDATED`/the translation namespace below — never
a third near-identical copy of `LegalDocument`'s markup.

## `LEGAL_LAST_UPDATED` — bumped by hand, never derived

```ts
// src/constants/legal.ts
// Bump the relevant date whenever that document's content actually changes —
// never derive it from the current date at render time.
export const LEGAL_LAST_UPDATED = {
  privacyPolicy: new Date("2026-09-24"),
  termsOfUse: new Date("2026-09-24"),
} as const;
```

`new Date()` at render time would silently relabel every visit as "updated
today," which is false. The date only means something if a human moves it
alongside the content edit it's documenting — treat it like a changelog
entry, not a timestamp.

## Translation shape

One block per document under `pages.legal`, each with `title`, `updated`
(takes `{date}`), and a `sections` array of `{ heading, body }` — read with
`.raw("sections")` since it's structured data, not a string to interpolate:

```json
"pages": {
  "legal": {
    "privacyPolicy": {
      "title": "Privacy Policy",
      "updated": "Last updated: {date}",
      "sections": [{ "heading": "...", "body": "..." }]
    },
    "termsOfUse": { "title": "...", "updated": "...", "sections": [] }
  }
}
```

## These routes stay public

Legal pages must be reachable by a signed-out visitor — add their path(s) to
`proxy.ts`'s matcher exclusion alongside `sign-in`/`error` (see auth.md), the
same way any other public route is excluded. A prefix match (`legal/.*`) is
easier to keep correct than listing each slug by hand, since a new document
added under `LEGAL_LAST_UPDATED` shouldn't also require a matcher edit to
stay reachable.


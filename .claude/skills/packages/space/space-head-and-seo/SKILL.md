---
name: space-head-and-seo
description: The static head/head export precedence and dedup rules (title/meta/link merged across the layout chain), the React-hoisting-vs-Preact-front-placement rendering difference, and the SEO helpers — buildCanonicalLink, buildHreflangLinks (with correct x-default), and the real robots.txt/sitemap.xml routes. Use when setting a page's title/meta/link, or building canonical/hreflang/robots/sitemap output.
---

Covers `<title>`/`<meta>`/`<link>` resolution and the SEO-helper functions
built on it. Structured data (JSON-LD) is explicitly **not** part of this
module — it's `@zanix/space-ui`'s `StructuredData` component, a UI-level
concern rendered inline in `component`, not a head-descriptor field.
File:line references point at `~/Documents/Development/ZanixLibraries/space`
— read the real code there before assuming this summary is still accurate.

## Golden rule (token savings)

- Resolve SEO helper inputs from `loader`, not from `head`'s function form —
  `head`'s function only ever receives the loader's own resolved data, never
  `ctx`, so anything needing `ctx.url` (`buildCanonicalLink`) must be
  computed in `loader` first.
- Trust the dedup rules (below) rather than manually de-duplicating `meta`/
  `link` entries by hand when composing a layout chain.

## Head resolution: `static head` and a layout's `head` export

```tsx
export default class ProductPage extends SpacePageController<{ id: string }> {
  static head = (data) => ({ title: data.product.name })
  loader = async (ctx) => ({ product: await getProduct(ctx.params.id) })
  component = ProductView
}
```

`static head` on a page: a plain `HeadDescriptor` object, or `(data) =>
HeadDescriptor` where `data` is exactly what `loader` resolved (the same
value `component` receives as props). A `layout.tsx` may export `export
const head` the same way, except its function form receives `params`
(never `loader` data — a layout has no `loader` of its own).

**Precedence**: a page wins over its nearest layout, up to the root layout —
checked field-by-field for `title`, per identity key for `meta`/`link`
(never whole-descriptor replacement).

**Deduplication**: `meta` is deduped by identity key (`name`, `property`, or
`httpEquiv` — whichever is present); a tag with none of the three is never
deduped against another. `link` is deduped by `rel`+`href`, plus `hreflang`
when set — so two `alternate` links (e.g. `x-default` and another language)
can legitimately share an `href` and both survive.

## Rendering order and the React-vs-Preact difference

The resolved head always renders **before** `component`'s own tree.

- **React 19**: hoists both into `<head>` in encounter order, so the
  head-descriptor's `<title>` becomes `document.title` (the first `<title>`
  in the document, per the HTML Living Standard).
- **Preact** (no hoisting): the resolved head is placed at the **front** of
  the real `<head>` after rendering — same outcome, different mechanism. A
  hand-authored `<title>` inside `component` under Preact renders wherever
  it sits in `<body>` and never becomes `document.title`.

A hand-authored JSX `<title>`/`<meta>`/`<link>` inside `component` is never
suppressed by either mechanism — the two coexist. The root layout never
cooperates with head placement and receives no head-related prop (see
`space-routing-and-rendering`'s document contract).

## SEO helpers

### `buildCanonicalLink(options): HeadLinkTag`

```ts
buildCanonicalLink({ url: ctx.url, keepParams: ['page'] })
// /products?page=2&sort=price&session=abc -> /products?page=2
```

Strips the query string by default, always resolves against `url.origin`
(never a separately-configured domain). `keepParams` names the query params
to preserve; every other one is dropped.

### `buildHreflangLinks(options): HeadLinkTag[]`

```ts
buildHreflangLinks({ url, lang, availableLangs, defaultLang })
```

One `alternate` link per `availableLangs` entry (always including a
self-reference for the current `lang`), plus one `x-default` entry.
`x-default` resolves to `{origin}/{defaultLang}{rest}` (the `defaultLang`
version of the **same** page, never the bare site root). Pure function — no
React/Preact/hook/context dependency, identical output under either
renderer.

### `robots.txt` — `buildRobotsTxt`/`registerRobots`

```ts
export default defineSpaceApp({
  name: 'storefront',
  robots: { rules: [{ userAgent: '*', disallow: ['/admin'] }], includeSitemap: true },
})
```

Config via `defineSpaceApp({ robots })` — omitted means the route is never
registered at all (same convention as `assetsDir`/`messagesDir`/`sitemap`).
A raw `string` is served byte-for-byte, no trailing newline added. A
`RobotsConfig` object auto-appends `Sitemap: {origin}/sitemap.xml` when
`sitemap` is also configured and `includeSitemap !== false` (a no-op if
`sitemap` isn't configured at all).

### `sitemap.xml` — `buildSitemapXml`/`registerSitemap`

```ts
export default defineSpaceApp({
  name: 'storefront',
  sitemap: [{ loc: '/products/widget', lastmod: '2026-08-15', alternates: [
    { lang: 'en', href: '/en/products/widget' },
    { lang: 'es', href: '/es/products/widget' },
  ] }],
})
```

`SitemapSource` is either an array or a `() => SitemapEntry[] |
Promise<SitemapEntry[]>`. **No implicit self-inclusion in `alternates`** —
include the URL's own language too, or `buildSitemapXml` only lists exactly
what's named.

Caching differs by source kind:
- **Array**: the exact same reference is kept for the process lifetime,
  never recomputed — mutating it after `defineSpaceApp()` returns is
  reflected on the next request. The XML string is still rebuilt per
  request (it needs the current origin).
- **Function**: called once, cached in memory for the process lifetime (same
  pattern as `loadMessages()`). The cached value is the resolved entries,
  never the final XML. **Bypassed entirely under `znx space dev`**, so edits
  are reflected next request, no restart. In production, a function
  source's freshness is only as good as the last process start — an
  accepted trade-off, not a bug; an app needing sub-restart freshness
  manages its own invalidation inside the function.

**Deliberate design choice, avoids a real footgun**: relative `loc`/`href`
values resolve against the request's own origin, explicitly to avoid "a
`SITE_DOMAIN`-style env var read independently in multiple places, silently
producing invalid relative `<loc>` values wherever one goes unset."
Redirected routes are never mixed into the same `<urlset>` as indexable
URLs.

## Checklist before changing head content or SEO output

- [ ] Does anything needing `ctx` (like `buildCanonicalLink`) get computed
      in `loader`, not inside `head`'s function form?
- [ ] Does a new `alternate`/hreflang entry include a self-reference for its
      own language — `buildHreflangLinks` never adds one implicitly for
      manually-built entries?
- [ ] Is a function-based `sitemap` source's staleness trade-off (only fresh
      as of last process start, in production) actually acceptable for this
      app — or does it need its own invalidation logic?
- [ ] Does anything structured-data-shaped (JSON-LD) route to
      `@zanix/space-ui`'s `StructuredData` instead of being added here as a
      head-descriptor field?

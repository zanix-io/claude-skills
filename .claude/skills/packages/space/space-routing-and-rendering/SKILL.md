---
name: space-routing-and-rendering
description: File-based page routing — layouts/loading/error segments, the document shell (root layout vs. default document), the renderer-agnostic document contract (DocumentModel), the not-found page, activation via @zanix/app/@zanix/server, and the @zanix/space/testing helpers. Use when adding/changing a route, layout, or the not-found page, or writing a test for one.
---

This is the core routing/rendering layer of `@zanix/space`. For selective
hydration, see `space-comets`; for client-side navigation, see
`space-orbit-navigation`; for guards/CSP/CSRF, see
`space-middleware-and-security`; for `<title>`/`<meta>`/`<link>`, see
`space-head-and-seo`. File:line references point at
`~/Documents/Development/ZanixLibraries/space` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- Copy the closest existing `page.tsx`/`layout.tsx` as a template rather than
  re-deriving the file-convention shape from prose — the conventions
  (`layout.tsx`/`loading.tsx`/`error.tsx`/`page.tsx`, nesting per directory
  level) are mechanical, not something to reason about from scratch per
  route.
- Verify a specific claim (e.g. `DocumentModel`'s exact shape) with a
  targeted grep in `src/modules/render/`/`src/modules/router/` rather than
  reading those modules end to end.

## File conventions per route segment

Under `routesDir` (default `'./routes'`), each directory level may contain:

| File | Role |
| --- | --- |
| `page.tsx` | The route itself. Exports a `SpacePageController` subclass with `loader`/`component`/`action` as separate, independently testable members. |
| `layout.tsx` | Wraps `page.tsx` and everything nested below it — never a route of its own. Nested per directory level; a page is wrapped by every ancestor `layout.tsx`, innermost first. |
| `loading.tsx` | Suspense fallback for its segment and everything nested under it. |
| `error.tsx` | Error boundary for its segment. |
| `not-found.tsx` | Renders for any unmatched route under this `routesDir` (see below). |

```tsx
// routes/products/layout.tsx
import type { LayoutProps } from '@zanix/space'

export default function ProductsLayout({ children }: LayoutProps) {
  return <section className='products'>{children}</section>
}
```

`LayoutProps` is renderer-neutral — the bare form (no type argument) is
correct for both React and Preact, since `children`'s default type
(`SpaceChildren`) is a structural type both `ReactNode` and
`ComponentChildren` accept. Name a renderer's own type explicitly
(`LayoutProps<ReactNode>`/`LayoutProps<ComponentChildren>`) only for
something `SpaceChildren` doesn't express.

**`error.tsx`'s real current limitation**: React's server renderer only
recovers a thrown error for content inside a `Suspense` boundary, which
`SpacePageController` always adds around a page, so a thrown error keeps the
response at a real `200` instead of breaking the whole document. The
fallback's own UI, however, only becomes visible after client hydration —
there is no visible fallback content during the initial server-rendered
response yet. Don't assume `error.tsx` renders anything visible server-side
today; it only protects the response from breaking.

## Document shell: root layout vs. default document

A root `routes/layout.tsx` is trusted, as-is, to render the actual
`<html>`/`<body>` document — same contract as Next.js's own App Router.
Nothing validates that it actually returns `<html>`/`<body>`.

```tsx
// routes/layout.tsx
import type { LayoutProps } from '@zanix/space'

export default function RootLayout({ children }: LayoutProps) {
  return (
    <html lang='en'>
      <head>
        <meta charSet='utf-8' />
        <meta name='viewport' content='width=device-width, initial-scale=1' />
      </head>
      <body>{children}</body>
    </html>
  )
}
```

With no root layout at all, `SpacePageController` wraps every page in a
minimal, spec-valid default document (`<!DOCTYPE html>`, UTF-8 charset,
responsive viewport meta) — a project is never left without a real document.
Global UI (header/footer/nav) belongs in this root layout; there's no
separate mechanism, since nested layouts already compose global + per-section
wrapping.

**Rule**: the root layout owns document structure only — it never receives
or renders anything head-related (no `<title>`/`<meta>` prop). `@zanix/space`
places resolved head content into the document itself, under either
renderer, independently of what the root layout returns (see
`space-head-and-seo`).

**Generator caution**: `zanix generate layout ''` writes the full document
shape shown above. Generating a bare `<div>` wrapper at the root instead
would silently replace a valid document with one missing doctype/`lang`/
charset/viewport.

## The document contract

```
page + layout chain + loader data
              ↓
        DocumentModel          ← resolved once: title/meta/link, css, theme, lang, PWA
       ↙            ↘
React serializer   Preact serializer
       ↘            ↙
        final document
              ↓
     renderer-agnostic validation
```

`DocumentModel` (`src/modules/render/document-model.ts`) is a renderer-agnostic
type carrying no React/Preact type at all; `resolveHead`
(`src/modules/router/head-descriptor.ts`) runs once per request; the
serializers (React vs. Preact, `src/modules/render/head-markup.ts`'s
`serializeHeadMarkup`) do no merging/deduplication/reordering of their own —
that's already done by the time they run.

React and Preact are two implementations of one contract, not two contracts:
same semantics (title/meta/link set, `lang`, stylesheet links) but not the
same bytes — attribute order, void-element closing, and whitespace legitimately
differ between them.

**PWA is orthogonal, not a renderer.** There's no "PWA renderer" — the real
matrix is `renderer × pwa` (4 combinations). PWA contributes a manifest link,
`theme-color`, and service-worker registration to whichever renderer's
document is being produced; the Web App Manifest and service worker
themselves are separate files, validated separately from the HTML document
(see `space-pwa`).

## Not-found page

```tsx
// routes/not-found.tsx
export default function NotFound() {
  return <h1>Page not found</h1>
}

export const head = { title: 'Page not found', meta: [{ name: 'robots', content: 'noindex' }] }
```

Optional `head` export, same convention as `layout.tsx`'s own `head` export
(see `space-head-and-seo`) — defaults to `{ title: 'Page not found' }` when
omitted. Goes through the same `DocumentModel`/head resolution as any other
page; there's no not-found-specific title/heading rule.

```ts
// main.ts
import { createNotFoundHandler } from '@zanix/space'
import { bootstrapServers } from '@zanix/server'

await bootstrapServers({
  ssr: { application: 'storefront', onError: createNotFoundHandler() },
})
```

`createNotFoundHandler()` is opt-in — `@zanix/space` never wires it up on
its own — and only ever handles an actual `404`; anything else falls through
to `@zanix/server`'s own default error response, unchanged. With no
`not-found.tsx` present, it renders a minimal built-in default instead of
failing.

**Security-relevant, deliberately opt-in flag**: `ssr.attachRequestToErrors:
true` on `bootstrapServers` is what makes `@zanix/server` hand the original
`Request` to `onError` at all — needed for the Orbit-aware branch (a missing
route reached via Orbit navigation gets just the outlet fragment instead of
a full document). It defaults to `false` because a `Request` can carry
`Authorization`/cookies, so it's never attached to `onError` unless a
consumer deliberately opts in. Without the flag, every `404` still gets the
full document; Orbit's client runtime degrades gracefully on any non-`ok`
fragment response via a plain `location.href` navigation — one wasted
round-trip, nothing broken.

## Activation

```ts
// main.ts
import spaceApp from './space.app.ts'
import { activateApps } from '@zanix/app/runtime'
import { bootstrapServers } from '@zanix/server'

await activateApps([spaceApp])
await bootstrapServers({ ssr: { application: 'storefront' } })
```

Activation is always `@zanix/app`'s/`@zanix/server`'s responsibility, never
`@zanix/space`'s own — `activateApps()` runs `loadRoutes()` internally (an
author never calls `loadRoutes()` directly) and runs `onStart`.
`bootstrapServers({ ssr: { application } })`'s `application` must match the
app's own `name`; `@zanix/space` never assumes a default Application.

## Testing pages

```ts
import { mockPageContext, renderPageForTest } from '@zanix/space/testing'
```

Always the `@zanix/space/testing` subpath, never the package root.

- **`mockPageContext<Params>(overrides?)`** — builds the exact `{request,
  url, params, csrfToken}` shape a `loader`/`action` receives, typed against
  the page's own `Params` generic. Unit level: exercises `loader` as a plain
  function, no rendering.
- **`renderPageForTest(Controller, params?, ctxOverrides?)`** — generic over
  the same `Params` (a page declared `SpacePageController<{ id: string }>`
  requires `{ id: string }` for `params`, not just any
  `Record<string, string>`). Instantiates `Controller`, calls its real
  `handleGet`, resolves once the streamed response has fully settled.
  Functional level: the real `loader → component → handleGet` pipeline,
  in-process, no HTTP.
- **`mockHandlerContext`** — the lower-level `HandlerContext` builder both of
  the above use internally; reach for it directly only when testing
  something below the page level (e.g. a custom `@Guard`).

## Checklist before adding/changing a route

- [ ] Does the file convention match the segment's real role (`page.tsx`
      never doubling as a `layout.tsx`, `error.tsx` not assumed to render
      visible fallback content server-side)?
- [ ] If a root `layout.tsx` is added/changed, does it own only document
      structure — no head-related prop, no assumption about what
      `@zanix/space` injects into `<head>`?
- [ ] Is `createNotFoundHandler()`/`attachRequestToErrors` wired
      deliberately, with the security trade-off of the latter actually
      considered — not copied from another project without checking whether
      this app's `onError` chain is safe to receive the raw `Request`?
- [ ] Does a new page's test use `mockPageContext` (unit) or
      `renderPageForTest` (functional) as appropriate, from
      `@zanix/space/testing` — not a hand-rolled context object?

---
name: space-orbit-navigation
description: Client-side navigation ("Orbit") — initOrbit's prefetch config, how a navigation fragment carries and dedupes the destination page's CSS (avoiding FOUC), renderToResponse/useRequestCache, readInitialState, the initialState serialization contract (and its real footguns with Date/Map/Set/circular refs), and the graceful-degradation guarantee. Use when configuring prefetch, passing data through initialState, debugging a flash of unstyled content on navigation, or debugging an Orbit navigation.
---

Orbit swaps a page's outlet fragment client-side instead of a full navigation
— every link still works as a real, full-page link with no JS at all. For
the routing/document layer Orbit swaps within, see
`space-routing-and-rendering`; for Comets (which Orbit's `persist` prop
interacts with), see `space-comets`. File:line references point at
`~/Documents/Development/ZanixLibraries/space` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- Trust the default prefetch config (hover/focus on, viewport off) unless a
  real UX reason calls for changing it — don't reconfigure `initOrbit()`
  speculatively.
- Check the serialization footguns table below before passing anything
  through `initialState` that isn't a plain, JSON-safe value — it's cheaper
  to check once than to debug a silently-dropped `Map` entry later.

## Setup

```ts
import { initOrbit } from '@zanix/space/client'
initOrbit()
```

Call once, alongside `hydrateComets()` (`space-comets`).

## Prefetch configuration

```ts
initOrbit() // default: hover/focus prefetch on, viewport prefetch off
initOrbit({ prefetch: { onViewport: true } })
initOrbit({ prefetch: { onHover: false, onViewport: true } })
initOrbit({ prefetch: false }) // disables prefetch entirely; navigation itself unaffected
```

Defaults: `onHover: true` (mouseenter/focusin, ~120ms debounce), `onViewport:
false` (IntersectionObserver, opt-in).

**Guards, silent by design**: prefetch is silently disabled when
`navigator.connection.saveData` is on or `effectiveType` is `'slow-2g'`/
`'2g'` — no error, it just doesn't fire. At most 4 concurrent prefetches; a
5th trigger is simply dropped, no queue, no retry. A failed prefetch is
evicted immediately so a subsequent click always gets a fresh fetch rather
than repeating the same failure.

**Escape hatch**: `data-orbit-hard` on an `<a>` forces a full navigation for
that link specifically.

## Graceful degradation

Any non-2xx fragment response (404/500/network failure) silently degrades to
a full navigation — a broken Orbit fetch is never a broken UX, only a
slightly slower one. HTTP caches in front of the app must key on the
`Vary: X-Znx-Space-Navigate` header too (set unconditionally by every
response `SpacePageController`/`createNotFoundHandler` produce), not just
rely on Orbit's own client runtime to always request the right variant.

**Current limitation**: nested layouts aren't preserved across sibling
routes yet — navigating `/products/1` → `/products/2` re-renders everything
under the root layout, not just the leaf.

## CSS during navigation

A fragment response carries every stylesheet the destination page needs
(its own `static styles` plus any Comet it renders) as real
`<link rel="stylesheet">` tags in its body — see `space-styling-and-theming`
for the `global`/page/comet delivery contract itself. `global` is never
repeated in a fragment — it's app-wide, already present since the initial
load.

Before completing a swap, the client extracts every `<link>` from the
fragment, dedupes by `href` against what the current document already has
**anywhere in it** (not just `<head>` — a Comet can leave its own `<link>`
in `<body>`), and inserts only what's missing into `<head>`, synchronously
and in order (`media` preserved), waiting for each to load
(`load`/`error`, or a 4s timeout that never rejects — the swap always
proceeds) before the visual swap happens. This is what avoids a flash of
unstyled content navigating into a page whose CSS the current document
doesn't have yet. Two overlapping navigations needing the same missing
stylesheet share one in-flight load instead of inserting a duplicate
`<link>` — there's no client-side registry of "what CSS exists" beyond
that; the manifest itself stays server-side, read fresh from each fragment.
A page whose CSS is already fully covered triggers none of this — no
`<link>` insertion needed at all, the common case.

## `renderToResponse` and `useRequestCache`

```tsx
import { renderToResponse, useRequestCache } from '@zanix/space/react'

function ProductView({ id }: { id: string }) {
  const product = useRequestCache(`product:${id}`, () => getProduct(id))
  return <h1>{product.name}</h1>
}

const response = await renderToResponse(<ProductView id='1' />, { initialState: { id: '1' } })
```

`useRequestCache` is **React-only** — it needs `Suspense`/`use()`. Under
`renderer: 'preact'` it throws immediately when called; use the page's
`loader` instead of a request-scoped cache hook there.

## `readInitialState`

```ts
import { readInitialState } from '@zanix/space/client' // never the root entry

const { id } = readInitialState<{ id: string }>() ?? {}
```

Import from `@zanix/space/client`, never the package root — importing from
root would risk bundling `react-dom/server` into client code.

## The `initialState` serialization contract — real footguns

`renderToResponse`'s `initialState` goes through one serialized global, not
scattered ad hoc globals. What actually happens to non-JSON-safe values:

| Value | Default behavior | With `serialization: { extendedTypes: true }` |
| --- | --- | --- |
| `undefined`, functions | Silently dropped | Same — never round-trips |
| `Date` | Becomes an ISO string, never revived to a real `Date` | Round-trips as a real `Date` |
| `Map`/`Set` | Serializes to `{}` — **every entry silently lost** | Round-trips correctly |
| Circular reference, `BigInt` | Fails outright | Fails outright — not fixed by this flag |

```ts
defineSpaceApp({ name: 'storefront', serialization: { extendedTypes: true } })
```

Opt-in, off by default, app-wide — enables `Date`/`Map`/`Set` round-tripping
through both `initialState` and Comet props, on both renderers. **Check this
table before passing a `Map`/`Set`/`Date` through `initialState` without
`extendedTypes` on** — a silently-emptied `Map` is a real, hard-to-notice bug
class, not a hypothetical.

A circular reference or `BigInt` in `initialState` makes `renderToResponse`
resolve a real `500` (calling `onError` if one is configured) rather than
throwing — a Comet's own bad prop, by contrast, throws a named error instead.
Don't assume the two failure modes look the same when debugging.

## Checklist before passing data through `initialState`

- [ ] Is every value plain-JSON-safe, or is `extendedTypes: true` actually
      configured if a `Date`/`Map`/`Set` is involved?
- [ ] Is anything circular or a `BigInt` excluded — neither is fixed by
      `extendedTypes`?
- [ ] Does prefetch configuration match a real UX need, rather than being
      left unconfigured by default or turned on everywhere speculatively?
- [ ] If a shared HTTP cache sits in front of this app, does it vary on
      `X-Znx-Space-Navigate`?

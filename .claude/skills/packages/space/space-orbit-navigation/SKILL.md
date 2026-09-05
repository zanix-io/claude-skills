---
name: space-orbit-navigation
description: Client-side navigation ("Orbit") — initOrbit's prefetch config, how a navigation fragment carries and dedupes the destination page's CSS (avoiding FOUC), how a fragment whose own resolved CSP differs from the active document's automatically falls back to a real navigation instead of applying under the wrong policy, getActiveCspNonce() for a Comet generating its own nonce'd content client-side, swapOutlet's own overlapping-navigation serialization, renderToResponse/useRequestCache, readInitialState, the initialState serialization contract (and its real footguns with Date/Map/Set/circular refs), and the graceful-degradation guarantee. Use when configuring prefetch, passing data through initialState, debugging a flash of unstyled content on navigation, debugging a CSP violation that only reproduces via an in-app link click, or debugging an Orbit navigation.
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

Any non-2xx fragment response (404/500/network failure), or a fragment whose
own resolved CSP differs from the currently active document's (see "CSP
during navigation" below), silently degrades to a full navigation — a broken
or CSP-unsafe Orbit swap is never a broken UX, only a slightly slower one.
HTTP caches in front of the app must key on the `Vary: X-Znx-Space-Navigate`
header too (set unconditionally by every response
`SpacePageController`/`createNotFoundHandler` produce), not just rely on
Orbit's own client runtime to always request the right variant.

**Current limitation**: nested layouts aren't preserved across sibling
routes yet — navigating `/products/1` → `/products/2` re-renders everything
under the root layout, not just the leaf.

**Current limitation**: `swapOutlet`'s `document.startViewTransition(swap)`
is the only View Transitions reference in this package, with no per-element
`view-transition-name` anywhere — so a `persist`-tagged Comet (its state
genuinely survives the swap, see `space-comets`) still gets visually
crossfaded as part of the whole outlet's default transition, indistinguishable
from a torn-down-and-recreated element. See `space-comets`'s own `persist`
section for the full detail; this note exists so a session investigating the
swap mechanism itself finds it here too, not only via `persist`.

**Real, confirmed, and mostly benign: `startViewTransition`'s `.ready`
promise rejecting is expected, not a failure signal.** Confirmed via live
instrumentation (wrapping `document.startViewTransition` to observe its
returned `ViewTransition`'s own promises): a single, ordinary navigation
reliably produces `.ready` rejecting with `InvalidStateError: Transition was
aborted because of invalid state` in the console, while `swap()` itself
completes and the destination renders correctly — the browser simply
skipped the visual cross-fade animation, which is unrelated to whether the
DOM update happened. Don't chase this specific console message as a bug on
its own.

**Fixed: overlapping `swapOutlet` calls are serialized, closing a real
"outlet ends up genuinely empty" gap.** Two navigation attempts on the same
outlet overlapping (e.g. a click that doesn't visibly register followed
immediately by a second one, before the first's transition has settled)
used to race their own synchronous DOM mutations against each other,
confirmed to leave the outlet's ENTIRE fragment content missing, not just a
scroll-position illusion — nothing tracked an in-flight
`document.startViewTransition(swap)` before allowing a second one to
start. `swapOutlet` (`orbit.ts`) now tracks the in-flight swap in a
module-level `pendingSwap`; a call landing while one is already in flight
awaits it FIRST, before doing anything else (including its own `fetch`),
so every navigation still completes, in the order triggered, rather than
one being silently dropped or corrupting the other's DOM — the trade-off
is a fast-following second click visibly landing on the first destination
for a moment before the second's own swap takes over. `pendingSwap` stays
`null` (not an always-resolved `Promise`) in the common, non-overlapping
case specifically so this adds no extra microtask tick there — a
`.then()` against an already-resolved promise still schedules one, which
would desync every fire-and-forget caller's own timing (`onClick`/
`onPopState` never await `swapOutlet`) for every navigation, not just an
overlapping one.

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

## CSP during navigation

**Real, fixed bug — the mechanism, so a future regression is recognizable.**
A document's active `Content-Security-Policy` is fixed at the navigation
that created it: no later `fetch()` response, regardless of its own
headers, is ever consulted by the browser to update it — a real browser
security-model constraint, not something `@zanix/space` itself could ever
work around client-side. Before the fix below, a destination page whose own
resolved CSP genuinely differed from the currently active one (a stricter
or looser per-page `Page({ headers: { csp } })`, or a guard-registered
`cspGuard()` that varies the policy per request rather than per route) still
got its fragment swapped in via Orbit — silently enforced against the WRONG,
still-active policy, even though the fragment's own response DID carry the
correct header for the destination.

**The fix, so it's clear this is already handled, not something to route
around by hand.** Every full-document render embeds its own resolved CSP,
normalized (its own per-request nonce replaced with a placeholder so two
navigations to the same page never look like "a different policy"), as a
`<meta name="x-space-csp-signature">` — see `csp-signature.ts`'s own module
doc (`normalizeCspSignature`/`withCspSignatureMeta`, shared by both
renderers so this behaves identically under `--renderer=react` and
`--renderer=preact`; never emitted into a fragment's own head, since a
fragment isn't a document). Before ever swapping a fetched (or prefetched —
`prefetch.ts` now carries a fragment's `Content-Security-Policy` header
alongside its body for exactly this) fragment into the live DOM, `orbit.ts`
compares that fragment response's own header, normalized the same way,
against the active document's embedded signature (confirmed via
`csp-signature.test.ts`'s own suite and `orbit.ts:203`). A genuine mismatch
degrades to a real navigation instead of swapping — the ONE thing that can
actually apply a different CSP correctly. A page with no CSP configured, or
navigating between two pages sharing the exact same resolved policy (the
overwhelming common case — only the nonce differs, and that's normalized
away), is completely unaffected: no extra round-trip, no behavior change.

This needs no configuration and no opt-in — it's automatic for every app.
`data-orbit-hard` on a specific link is no longer a required workaround for
this (it still exists for other reasons); a per-page CSP difference is now
detected and handled generally, for every link, not just ones an author
remembered to annotate.

**A Comet that bakes its own `cspNonce` prop into freshly-generated
client-side content needs `getActiveCspNonce()` instead, never the prop
itself.** The comparison above deliberately normalizes the nonce OUT of the
check (`'nonce-<value>'` → `'nonce-*'`), which is correct for its own job
(two pages sharing the same CSP *shape* shouldn't hard-navigate just
because their nonces differ) — but it means the comparison has no
visibility into, and can't protect, a Comet that takes its
`PageContext.cspNonce` prop (the documented `space-middleware-and-security`
pattern) and uses it to build NEW nonce'd markup at hydration time (e.g. a
generated `<style nonce={nonce}>` inserted into a sandboxed iframe's
`srcDoc`). That prop reflects the fragment's OWN, separately-minted nonce —
never what the still-active top document (Orbit only swaps the outlet,
never the whole document) is actually enforcing; baking it into new inline
content works by accident on a hard reload (same response mints both
values) and produces a real, reproducible CSP violation on every
Orbit-navigated visit otherwise. A Comet that just forwards `nonce`
straight through to something like `@zanix/space-ui`'s Modal/Drawer (which
render their own `<style nonce>` once, at that component's first mount)
isn't affected the same way — the risk is specific to a Comet generating
brand-new nonce'd content on ITS OWN, from a prop.

```ts
import { getActiveCspNonce } from '@zanix/space/client'

const nonce = getActiveCspNonce() // the ACTIVE document's own nonce, right now — or undefined
```

`getActiveCspNonce()` (`active-nonce.ts`) reads the nonce a real, parsed
element on the page already carries — the `.nonce` IDL property, never
`getAttribute('nonce')` (a real browser hides a nonce's own content
attribute from `getAttribute`/`outerHTML` once inserted, for security, but
the IDL property still returns the real value for a PARSER-inserted
element) — always accurate regardless of how the current document was
reached. `undefined` on a page with no nonce-based CSP configured at all.
Same technique `comet-persist-transition.ts` already used internally for
the identical problem (giving its own dynamically-created `<style>`
element the currently-enforced nonce), now extracted into this one shared
helper instead of two separate hand-rolled copies.

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

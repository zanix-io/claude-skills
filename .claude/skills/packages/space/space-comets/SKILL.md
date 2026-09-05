---
name: space-comets
description: Selective hydration — the 'use comet' directive, defineComet, the comet manifest and why it exists, hydration timing (visible/load/only), the persist prop's bounded cache, and the 'server-only' build-time enforcement directive. Also covers the ready-made, headless Comet catalog this package ships (FormDraftPersistence, SubmitGuard, ScrollRestoration, UnsavedChangesGuard, NetworkStatus, ManagedForm) and why none of them render the `<form>`/accept children. Use when building or reviewing a Comet (an islands-architecture hydration boundary), or before hand-rolling a form-draft/scroll-restoration/double-submit/unsaved-changes/online-status behavior a page author might reasonably assume already exists.
---

Comets are `@zanix/space`'s selective-hydration mechanism (an islands
architecture). For the framework's routing/document layer around them, see
`space-routing-and-rendering`; for client-side navigation, see
`space-orbit-navigation`. File:line references point at
`~/Documents/Development/ZanixLibraries/space` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- A comet file needs exactly three things (directive, named export,
  `import.meta.url`) — verify a new comet against that checklist directly
  rather than re-deriving the requirement from prose each time.
- Confirm the client entry imports the barrel matching the app's own
  `renderer` (React vs. Preact) once per app, not per comet — it's a
  build-time-enforced, app-wide setting, not a per-file concern.
- Before hand-rolling form-draft recovery, double-submit prevention, scroll
  restoration, an unsaved-changes warning, or an online/offline indicator,
  check "Ready-made Comets this package ships" below — a real, shipped
  catalog exists for exactly these, not just architectural guidance for
  building one from scratch.

## A comet file, and its three real requirements

```tsx
// comets/counter.tsx
'use comet'
import { defineComet } from '@zanix/space/comet'

export function Counter({ initial }: { initial: number }) {
  const [count, setCount] = useState(initial)
  return <button onClick={() => setCount((c) => c + 1)}>{count}</button>
}

export default defineComet(Counter, import.meta.url)
```

**Always `@zanix/space/comet`, never the root `@zanix/space` barrel** —
`defineComet`/`loadCometManifest`/`resolveCometModuleUrl` are deliberately
NOT re-exported from `.` (`mod.ts:197-213`, `comets/mod.ts`'s own module
doc). The root barrel also carries genuinely server/dev-only exports
(`defineSpaceApp`, `SpaceDevSocket`'s real TC39 decorators, which a normal
Vite transform can't even parse) — a Comet's own client bundle resolving
`.` as a whole has no way to pull in only `defineComet`'s narrow slice, so
`import { defineComet } from '@zanix/space'` is a real, confirmed **hard
client-build failure**, not a style preference or a deprecated-but-working
path.

1. **`'use comet'`** — the first statement in the file, same grammar slot as
   React Server Components' `'use client'`. This is how `cometPlugin` finds
   the file and forces it into its own build output chunk; no particular
   directory is required.
2. **A *named* function export** (`Counter`, never anonymous) — `defineComet`
   reads `Counter.name` for the client to import back out of the module once
   fetched. The `defineComet(...)`-wrapped version becomes the file's
   *default* export instead, so the two don't collide. **The real failure
   mode when this is missed (e.g. `function Counter(...)` declared but never
   `export`ed) is silent, not a build error**: the page server-renders fine,
   the boundary's initial HTML looks correct — then the client's dynamic
   `import()` can't find the named export, hydration never happens, and
   anything interactive (a button, a form) simply vanishes from the DOM
   after first paint with nothing thrown to the console. Confirmed via a
   real repro, not theoretical — if a Comet's content disappears right
   after the page loads rather than failing to appear at all, check the
   export first.
3. **`import.meta.url`, always at this exact call site** — correlated at
   build/serve time to the hashed client build URL (see "Why the manifest
   exists" below).

## Vite setup

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import { cometPlugin, spacePlugin } from '@zanix/space/vite'

export default defineConfig({ plugins: [spacePlugin(), cometPlugin()] })
```

Required so a comet's file is built as its own separate output chunk rather
than inlined into whatever page server-renders it.

**Dev-only caveat — a Comet's own `*.module.css` import flashes unstyled
before styled, never in production**: the class names in the SSR'd HTML are
correct from byte one either way, but in dev the actual CSS *rules* only
land once the Comet's own client chunk executes and Vite's dev server
injects a JS-driven `<style>` tag for it (`dev-css-hrefs.ts`) — that always
happens after first paint, even for `comet='load'`, producing a real,
visible unstyled→styled flash. In a real build, `cssPlugin` instead emits a
genuine `<link>` for that same CSS, so nothing depends on any client JS
running — no flash there. A page's own `static styles`/`globalCss` never has
this problem in dev either, since those are already served via a real
`<link>` (with a `?direct` suffix, no manifest, no build step) with zero
dependency on hydration. Nothing to fix here — it's inherent to how a dev
server can serve JS-imported CSS without a bundling pass; just don't
mistake it for a real styling bug while iterating in dev.

## Manifest loading and why it exists

```ts
// main.ts
import { loadCometManifest } from '@zanix/space/comet'

await loadCometManifest('./dist/client/comets-manifest.json')
```

The same comet source file is evaluated twice — once server-side (a direct
Deno import producing real HTML) and once inside the client build (its own
bundled chunk) — two separate module instances/environments. `import.meta.url`
read server-side doesn't by itself resolve to the client chunk's hashed URL.
`cometPlugin` writes `comets-manifest.json` (source file → real built URL)
during the client build; `loadCometManifest` reads it back at startup so
`defineComet` resolves the right URL per request. **Dev-mode exception**: no
manifest is needed in development — Vite's dev server serves every project
file at its own root-relative path, so `defineComet` derives a working URL
directly, with no build step.

## Using a comet, and hydration timing

```tsx
import Counter from '../comets/counter.tsx'

<Counter initial={0} comet='visible' /> // hydrates once scrolled into view
<Counter initial={0} /> // hydrates immediately — comet defaults to 'load'
```

`comet` prop values: `'visible'`, default `'load'` (hydrates immediately),
and `'only'` (mounts fresh client-side via `createRoot`, never
`hydrateRoot`/SSR at all — for browser-only content).

## Client entry: React vs. Preact barrel — pick the one matching the app's renderer

```ts
import { hydrateComets } from '@zanix/space/client' // React barrel
// or, for renderer: 'preact' apps:
import { hydrateComets } from '@zanix/space/client/preact'

hydrateComets()
```

**`@zanix/space/client` is the React barrel; a `renderer: 'preact'` app must
import `@zanix/space/client/preact` instead** — same exports/signatures,
Preact's `hydrate`/`render` under the hood vs. React's `hydrateRoot`/
`createRoot`. Only ever import one, never both. Getting this wrong is a real
silent-failure mode: the page server-renders correctly, every comet boundary
and all its content appears in the DOM, nothing throws — and no comet is
ever interactive. `spacePlugin({ renderer })` fails the client build with an
explicit error if the entry imports the wrong barrel, which is what keeps a
mismatch from ever reaching a browser — confirm the client entry's import
matches the app's own `renderer` before assuming this is safe by
construction.

## `persist`: state across an Orbit navigation

```tsx
<Counter initial={0} comet='visible' persist='home-counter' />
```

**Opt-in, never automatic** — a Comet with no `persist` prop is destroyed
and recreated fresh on every Orbit navigation, exactly like any other
comet; nothing persists until an author explicitly passes a stable string
key (`define-comet.ts`'s own `CometBoundary` destructures `persist` with no
default, so it's `undefined` unless given). Passing `persist` keeps that
comet's real DOM node (and component state) alive across an Orbit swap
instead of tearing down/re-hydrating fresh — e.g. an in-progress form or an
open dropdown.

**Bounded-cache caution**: retained per `persist` key, bounded to the 5 most
recently used (`comet-persistence.ts`'s `liveCache`, a plain LRU — eviction
removes exactly the ONE oldest entry once a 6th key is added, never more).
Beyond the cap, or a mismatched comet (a different module+export)
reappearing under a reused key, is silently discarded. Don't assume
`persist` guarantees indefinite retention, or that reusing a `persist` key
across genuinely different comets is safe.

**Verifying retention costs a touch — checking IS navigating.** There's no
read-only way today to ask "is this key currently retained?" without
navigating to a page that renders it, and that navigation itself re-touches
the LRU (a real API gap — `RetainedCometCache`'s own `has(key)` already
exists internally, read-only, never consuming an entry, but neither it nor
the production `liveCache` singleton is exported from `comet-persistence.ts`
for a consumer or debugging code to call). **This produces a real,
easy-to-fall-into testing illusion**: checking several keys in
ascending/insertion order right after the cache fills up makes it look like
"everything got wiped" — each check-navigation evicts the very next key
you're about to check, cascading. Check retained keys **newest-to-oldest**
instead (the key you just navigated away from last) to see the true state;
confirmed via a live, careful re-test that exactly one entry evicts per
insert past the cap, never more, matching the LRU's own `while (size >
limit)` guarantee.

**Known real gap, already tracked, don't rediscover it**: a persisted
Comet's surviving DOM node still visibly flashes/crossfades on every Orbit
navigation, because nothing in `@zanix/space` gives it its own
`view-transition-name` (`orbit.ts`'s `document.startViewTransition(swap)`
is the only View Transitions reference in the whole package, with no
per-element name anywhere) — the underlying state genuinely survives, only
the transition's own visual treatment doesn't distinguish a preserved
element from a destroyed-and-recreated one.

## `'server-only'`: build-time-only enforcement

```ts
// db/client.ts
'server-only'
export function query() { /* ... */ }
```

Same directive-prologue mechanism as `'use comet'`. `cometPlugin` **fails the
build** (a fatal error, not a warning) if a `'server-only'`-marked module is
ever reachable from a Comet, even transitively — it prints the exact import
chain. **Nothing here adds a runtime check to the shipped bundle** — the
enforcement is only during `cometPlugin`'s own build step, so a change to
the build pipeline that skips `cometPlugin` would silently lose this
protection.

## Comet presentation override

A Comet is composed as part of a Zanix App manifest, so it can resolve its
own className/style via `@zanix/app`'s `resolveBehavior()` — see
`@zanix/app`'s own docs, "Style-only overrides." The one real precondition:
the Comet's own author must opt in by adding that call — this is never
retroactive for a comet that doesn't already use it.

## Ready-made Comets this package ships

`@zanix/space/comet/react`/`@zanix/space/comet/preact` (a NAMED import
ONLY — neither subpath has a default export the way one author's own
`comets/counter.tsx` does; more than one Comet lives in each barrel, so
there's no single "the" default) ship six hook-free, `null`-rendering
Comets, each a thin `useEffect` wrapper around a renderer-agnostic core
primitive exported from `@zanix/space/comet` itself:

| Comet | Primitive | Does |
| --- | --- | --- |
| `FormDraftPersistence` | `attachFormDraftPersistence` | Session/local-scoped `<form>` draft recovery across a refresh/navigate-away. |
| `SubmitGuard` | `attachSubmitGuard` | Stops a second real `<form>` submission from reaching the server. |
| `ScrollRestoration` | `attachScrollRestoration` | Window/container scroll-position recovery across a refresh or Orbit navigation. |
| `UnsavedChangesGuard` | `attachUnsavedChangesGuard` | Native `beforeunload` "leave site?" warning for a dirty `<form>`. |
| `NetworkStatus` | `attachNetworkStatus` | Live `navigator.onLine`, written as a `data-network-status` attribute (a Comet's props can't carry a callback). |
| `ManagedForm` | `attachManagedForm` | Composes the three form-level behaviors above under one `formId`. |

Full API/options for each: `docs/comets.md`'s own "Form draft persistence"
through "Composing form behaviors" sections — not restated here.

**Why none of them render the `<form>`/accept `children`, and neither
should a new one modeled on them**: a Comet's own props must be plain JSON
(`defineComet`'s `stringifyForWire` call, `define-comet.ts`). A component
that ALSO needs arbitrary field markup as `children` — closures, event
handlers, none of it JSON-serializable — can't be one hydratable boundary.
Every one of these instead attaches behavior to an EXTERNALLY-rendered,
ordinary `<form id="...">` by `id`, exactly like `persist`'s own
identity-by-string-key convention above — the shell stays plain
server-rendered markup, only the behavior is a Comet.

**A raw DOM write is invisible to a React/Preact-controlled field —
dispatch a real event after it, don't assume `.value =` is enough.**
`FormDraftPersistence`'s own restore step (`form-draft-persistence.ts`)
writes `.value`/`.checked` directly, then dispatches a real, bubbling
`input` (text-like) or `change` (`checkbox`/`radio`/`<select>`) event on
that same field — because `@zanix/space-ui`'s own `Input`/`Select`/
`RadioGroup` always track a `value` internally at the DOM level even when
the page author never passes one, so the raw write alone gets silently
reverted on that field's next re-render. The dispatched event is what
lets such a field's own `onChange`/`onInput` handler pick it up and sync
state, the same path a genuine keystroke/click already takes. Worth
knowing before writing ANY code that sets a form field's value from
outside its owning framework's own render cycle, not just inside this
package.

## Checklist before adding/changing a Comet

- [ ] Directive, named export, and `import.meta.url` all present and correct
      (missing any one silently breaks manifest correlation or client-side
      resolution, not always loudly — a missing `export` in particular
      renders fine server-side and only fails silently on hydration)?
- [ ] Does `defineComet`/`loadCometManifest` import from `@zanix/space/comet`
      — never the root `@zanix/space` barrel, which is a real hard build
      failure for a Comet's own client bundle, not just a lint nit?
- [ ] Is `comet='visible'`/`'load'` (default)/`'only'` chosen deliberately
      for the actual UX need, not left at the default without thinking about
      it?
- [ ] Does the client entry import the barrel matching this app's
      `renderer`?
- [ ] If `persist` is used, is the key genuinely stable across navigations
      for the SAME comet — not reused across different comets, and not
      relied on beyond the 5-entry cache?
- [ ] Testing `persist` retention by navigating and checking? Check keys
      newest-to-oldest, not insertion order — each check-navigation is
      itself a touch that can evict the next key you're about to check.
- [ ] Does anything server-only reachable from this comet's own imports need
      a `'server-only'` marker, so a future refactor can't silently leak it
      into the client bundle?
- [ ] Building a new ready-made Comet meant to ship from this package
      itself (not a consumer app's own one-off)? Does it stay `null`-
      rendering, attaching to an externally-rendered element by `id` rather
      than accepting `children` — the same JSON-props constraint every
      existing one already respects?

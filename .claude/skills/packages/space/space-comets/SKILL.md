---
name: space-comets
description: Selective hydration — the 'use comet' directive, defineComet, the comet manifest and why it exists, hydration timing (visible/load/only), the persist prop's bounded cache, and the 'server-only' build-time enforcement directive. Use when building or reviewing a Comet (an islands-architecture hydration boundary).
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

## A comet file, and its three real requirements

```tsx
// comets/counter.tsx
'use comet'
import { defineComet } from '@zanix/space'

export function Counter({ initial }: { initial: number }) {
  const [count, setCount] = useState(initial)
  return <button onClick={() => setCount((c) => c + 1)}>{count}</button>
}

export default defineComet(Counter, import.meta.url)
```

1. **`'use comet'`** — the first statement in the file, same grammar slot as
   React Server Components' `'use client'`. This is how `cometPlugin` finds
   the file and forces it into its own build output chunk; no particular
   directory is required.
2. **A *named* function export** (`Counter`, never anonymous) — `defineComet`
   reads `Counter.name` for the client to import back out of the module once
   fetched. The `defineComet(...)`-wrapped version becomes the file's
   *default* export instead, so the two don't collide.
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

## Manifest loading and why it exists

```ts
// main.ts
import { loadCometManifest } from '@zanix/space'

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

Keeps a comet's real DOM node (and component state) alive across an Orbit
swap instead of tearing down/re-hydrating fresh — e.g. an in-progress form
or an open dropdown.

**Bounded-cache caution**: retained per `persist` key, bounded to the 5 most
recently used. Beyond the cap, or a mismatched comet (a different
module+export) reappearing under a reused key, is silently discarded. Don't
assume `persist` guarantees indefinite retention, or that reusing a
`persist` key across genuinely different comets is safe.

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

## Checklist before adding/changing a Comet

- [ ] Directive, named export, and `import.meta.url` all present and correct
      (missing any one silently breaks manifest correlation or client-side
      resolution, not always loudly)?
- [ ] Is `comet='visible'`/`'load'` (default)/`'only'` chosen deliberately
      for the actual UX need, not left at the default without thinking about
      it?
- [ ] Does the client entry import the barrel matching this app's
      `renderer`?
- [ ] If `persist` is used, is the key genuinely stable across navigations
      for the SAME comet — not reused across different comets, and not
      relied on beyond the 5-entry cache?
- [ ] Does anything server-only reachable from this comet's own imports need
      a `'server-only'` marker, so a future refactor can't silently leak it
      into the client bundle?

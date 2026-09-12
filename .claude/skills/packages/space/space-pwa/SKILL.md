---
name: space-pwa
description: PwaConfig (author-facing icon/manifest/offline config via defineSpaceApp({ pwa })), loadPwaBuildOutput (the build-OUTPUT-facing runtime wiring, auto-loaded by defineSpaceApp({ clientBuildDir }) — a manual main.ts call is only for the narrower case of not using clientBuildDir), and the network-first-navigation/cache-first-everything-else service worker — real icon resizing, a computed Web App Manifest, and a small custom service worker, all derived from ONE config, never a separately-configured pwaPlugin. Use when configuring PWA installability or debugging a stale/missing icon, manifest field, or service worker.
---

Covers `@zanix/space`'s PWA support — real icon resizing, a computed Web App
Manifest, and a small custom service worker. For image optimization in
general (a separate pipeline from PWA icons), see `space-assets-and-media`.
File:line references point at `~/Documents/Development/ZanixLibraries/space`
— read the real code there before assuming this summary is still accurate.

## Golden rule (token savings)

- `PwaConfig` (the `defineSpaceApp({ pwa })` parameter) is the ONLY thing an
  author configures — it never takes a build-output path. Don't add
  `iconsDir`/`swPath`-shaped fields; they don't exist and never did (checked
  the full git history of `src/typings/pwa.ts` — not a removed feature).
- Trust the default icon sizes (192/512) unless a real installability
  requirement calls for more — don't add maskable-icon support by hand; it's
  deliberately not built (see below).

## Configuration: `PwaConfig`, and wiring the build output at runtime

```ts
// space.app.ts — author-facing config: identity, icon, offline behavior,
// AND the build output dir — `clientBuildDir` makes `setup()` (which runs
// when `activateApps()` runs) call `loadPwaBuildOutput` itself
import { defineSpaceApp } from '@zanix/space'

export default defineSpaceApp({
  name: 'storefront',
  clientBuildDir: './dist/client', // the client build's own output dir
  pwa: {
    name: 'Storefront',
    themeColor: '#2563eb',
    offlineFallback: '/offline',
    icon: './public/icon-source.png', // required whenever `pwa` is configured at all
  },
})
```

`PwaConfig` deliberately contains only what an author wants to express —
identity/icon/behavior — **never a build-output path**: icons and the
service worker are generated at `zanix space build` time into whatever
output directory the build actually used, and the runtime discovers that
directory itself via `loadPwaBuildOutput`. `defineSpaceApp({ clientBuildDir })`
already calls `loadPwaBuildOutput` (and six other loaders) for a production
`main.ts` automatically — see `space-assets-and-media`'s "Wiring the client
build output at runtime" for the shared mechanism and the full loader list.
**An author never calls `pwaPlugin` directly** — `zanix space build`
composes it internally from the same `PwaConfig` (via
`resolvePwaPluginOptions`), the same way it composes `cssPlugin`/
`cometPlugin`. `pwaPlugin` is still exported from `@zanix/space/vite` for an
advanced/custom build pipeline that bypasses the CLI entirely, but that's
not the documented path.

Calling `loadPwaBuildOutput` directly in `main.ts` is only for an app that
doesn't set `clientBuildDir` at all (e.g. a custom build layout the option
doesn't fit):

```ts
// main.ts — only when NOT using `clientBuildDir`; after activateApps(),
// before bootstrapServers()
import { loadPwaBuildOutput } from '@zanix/space'

loadPwaBuildOutput('./dist/client') // must match the client build's real output dir
```

**Real footgun, whichever path wires it**: the directory passed to
`loadPwaBuildOutput` (directly, or via `clientBuildDir`) must be the SAME
directory the client build actually wrote to (whatever `zanix space build`'s
`--out-dir` used) — build-time (Vite/Node) and request-time (the deployed
Deno server) have no shared memory to enforce this automatically. A missing
or wrong value isn't an error: icon/service-worker routes are simply never
registered, and `/manifest.webmanifest` alone still works either way, since
it needs no built file at all. `clientBuildDir` is also skipped entirely
under `znx space dev` (a stale on-disk build would otherwise resolve to old
hashed output) — see `space-assets-and-media` for why.

`manifest.webmanifest`, icons, and `sw.js` are registered as real, explicit
routes — the same underlying route-registration mechanism `Page()` uses
internally, not a static-file convenience.

## Icons

Default sizes generated: only `192`/`512` — the two Chrome installability
criteria check; each manifest icon entry carries no `purpose` field, which
browsers treat as `"any"`. **Maskable icons deliberately withheld, not a
gap to fix by hand**: a naive resize of an arbitrary square source would
crop real content under Android's own mask shapes — a real correctness
footgun avoided by omission rather than by generating something wrong.

## Service worker

Fetch strategy: **network-first for navigations** (falls back to cache,
then to `offlineFallback` if set), **cache-first for everything else**.
Precaches real hashed CSS scanned directly from the build output, plus the
`offlineFallback` route if configured.

**Deliberate implementation detail, not from `cssPlugin`'s manifest**: CSS
precache is scanned directly from build output rather than read from
`cssPlugin`'s own `css-manifest.json`, because Rollup gives no ordering
guarantee between two same-priority plugins' `generateBundle` hooks —
relying on the manifest here would be a race, not a bug in the manifest
itself.

## Not implemented yet

`protocolHandlers`/`fileHandlers`/`shareTarget`/`push` (Tier-2, mostly
Chromium-only manifest fields), and maskable icons (see above — a
deliberate omission, not a gap).

## Checklist before changing PWA configuration

- [ ] Does `clientBuildDir` (or, for an app not using it, `main.ts`'s own
      `loadPwaBuildOutput` argument) genuinely match the client build's real
      output directory — checked directly, not assumed consistent?
- [ ] Is `icon` set whenever `pwa` is configured at all — a PWA can't be
      installed without a real icon?
- [ ] Is a maskable-icon request routed to a real design asset instead of a
      naive resize of the existing square source?
- [ ] Does `offlineFallback` genuinely need to be set — an app with no
      offline fallback (or no service worker output registered at all) is a
      valid, supported configuration, not an oversight?

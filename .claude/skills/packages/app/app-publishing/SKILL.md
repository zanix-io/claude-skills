---
name: app-publishing
description: Distributing a defineZanixApp() as a JSR package for a different team's host to install — explicit-import consumption (no discovery mechanism), factory-vs-pre-built-constant, and what's honestly not implemented yet (rootDir/package loading, version compatibility checks). Use when publishing your own Zanix App as a package, or consuming one someone else published.
---

Covers distributing a Zanix App to a different team's host, as opposed to
authoring/composing one inside your own process (`app-manifest-and-composition`).
File:line references point at `~/Documents/Development/ZanixLibraries/app`
— read the real code there before assuming this summary is still accurate.

## Golden rule (token savings)

- A published app is consumed via an explicit import, always — don't look
  for a discovery/registration mechanism that doesn't exist yet.
- Decide factory vs. pre-built constant once, based on whether the
  manifest depends on host-supplied data — don't re-derive this per app.

## Publishing: a plain JSR package

```ts
// my-app/mod.ts
import { defineZanixApp } from '@zanix/app'

export const reviewsApp = defineZanixApp({
  name: 'reviews',
  dependencies: { database: { type: 'mongo', required: true } },
  routes: true,
})
```

```jsonc
// my-app/deno.jsonc
{ "name": "@my-scope/reviews-app", "version": "1.0.0", "exports": { ".": "./mod.ts" } }
```

```sh
deno publish
```

The manifest returned by `defineZanixApp()` is a plain JS value with no
JSR-specific awareness — publishing is ordinary JSR package publishing,
nothing `@zanix/app`-specific about the mechanics themselves.

## Consuming: explicit import, always

```ts
import Zanix from '@zanix/core'
import { reviewsApp } from 'jsr:@my-scope/reviews-app@^1.0.0'

await Zanix.start({
  apps: {
    reviews: {
      definition: reviewsApp,
      server: { rest: { port: 4000 } },
      uses: [{ slot: 'database', resourceName: 'mongo' }],
    },
  },
  resources: { mongo: { type: 'mongo', options: { uri: 'mongodb://localhost' } } },
})
```

**No discovery mechanism exists today**: the host must import the package
explicitly and pass the definition object by reference — there's no way to
hand `Zanix.start()` a bare package name/specifier string and have it load
the app for you. `@zanix/app` also has no schema/introspection mechanism
for a third party to discover what a published app requires —
documenting `dependencies` and `config` is the publishing package's own
README's job, not something this package generates for you.

A published app can also run in isolation before/without a host, via its
own `.serve()` (`app-manifest-and-composition`) — useful for the publishing
team's own local testing.

## Factory vs. pre-built constant — deciding which shape to export

```ts
// Pre-built constant — the manifest is fully knowable at author time; nothing a host
// supplies changes what it declares.
export const fooApp: ZanixAppDefinition = defineZanixApp({ name: 'foo', ... })

// Factory — the manifest's shape depends on options only the HOST knows (a deployment's
// own credentials, which optional capabilities to register, per-instance identity).
export function defineFooApp(options: FooAppOptions): ZanixAppDefinition {
  return defineZanixApp({ name: 'foo', /* ...built from options... */ })
}
```

Export a factory when the manifest genuinely varies per host; export a
constant when it doesn't. `@zanix/space`'s `defineSpaceApp()` and
`@zanix/admin`'s `defineAdminHubApp(options)`/`defineLocalAdminApp()` are
real precedents for this exact choice, already made in this ecosystem.

**Real footgun**: `defineZanixApp()` is never memoized across calls — two
different hosts calling the same factory with different options must get
two independently normalized manifests, never one shared, mutated
instance. Don't cache a factory's return value across distinct host
configurations expecting reuse to be safe.

## What's honestly not implemented yet

- **`rootDir`/`package` fields are stored but never read.** `registerApp`
  keeps both on the manifest, but there's no string-based auto-import
  mechanism yet — this "is not a limitation that requires cleverness to
  route around," per the doc itself: use explicit imports, don't try to
  build a workaround.
- **`version` is stored only** — nothing validates cross-app compatibility
  against it today. Don't rely on the manifest's own `version` field for
  anything a host needs to act on yet; the real version of record is the
  JSR package version itself. (Contrast: Remote Resource Binding's
  `requiredVersion` DOES check `version` — but only within the same
  composition, see `app-remote-calls-and-control-plane`. That's a
  different, narrower mechanism than general cross-app compatibility
  validation.)

## Checklist before publishing or consuming a Zanix App package

- [ ] Does the published package's own README document `dependencies` and
      `config` explicitly — since `@zanix/app` itself provides no
      discovery/introspection for a consuming host to find this?
- [ ] Is the exported shape (constant vs. factory) chosen because the
      manifest genuinely does or doesn't vary per host — not by habit?
- [ ] If the manifest is exported as a factory, is it re-invoked per host
      configuration rather than cached, given it's never memoized
      internally?
- [ ] Does anything depend on `version`/`rootDir`/`package` doing more than
      they currently do — being stored, not acted on?

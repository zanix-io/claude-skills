---
name: space-ui-icons
description: @zanix/space-ui's optional default icon catalog (CatalogIcon, 17 curated glyphs) and the createCatalogIcon factory for building your own — licensing/attribution, independence from theme, and a real svgo gotcha when optimizing a multi-symbol sprite. Use when using/extending the default catalog, building a project-specific icon catalog, or optimizing an SVG sprite for this package's convention.
---

File:line references point at `~/Documents/Development/ZanixLibraries/space-ui`
— read the real code there before assuming this summary is still accurate.

## Golden rule (token savings)

- The 17-name closed union and the factory's call signature are small and
  stable — confirm them against this skill's own listing rather than
  re-reading source for routine use.

## `CatalogIcon`: a thin resolver over `Icon`, never a second icon system

```
name → CATALOG_VIEWBOX[name] → { href, viewBox } → Icon
```

`Icon`'s own contract (`href`/`name`/`viewBox`) never changes — `CatalogIcon`
resolves `name` to the catalog's own `viewBox` (a plain object lookup, no
`Map`, no fetch, no I/O) and calls the real `Icon` with the result. `href`
is still the caller's to provide — `CatalogIcon` never resolves, imports, or
assumes a location for the sprite file itself. Importing this package
without ever referencing `CatalogIcon` costs nothing — no CSS, no asset, no
network call, no side effect anywhere in the module graph.

`CatalogIconName` is a **closed TypeScript union**, not a bare `string` —
`spinner`, `close`, `gear`, `phone`, `envelope`, `arrow-up`, `arrow-down`,
`arrow-left`, `arrow-right`, `map-location-dot`, `search`, `check`, `plus`,
`minus`, `triangle-exclamation`, `circle-info`, `circle-check`. Passing an
unknown name is a compile-time error, never a silently broken `<use
href="...#undefined">` at runtime. No brand/social icon is in this list —
Font Awesome's own license restricts brand/social marks to representing the
company they refer to (a trademark concern, not just copyright); this
catalog is scoped to generic UI glyphs only, by design. `SocialNetworks`
handles brand icons correctly today by having the caller supply each
network's own logo, never bundling one.

## Building your own catalog with `createCatalogIcon`

```
createCatalogIcon: (h, viewBoxByName) => (props) => E
```

`CatalogIcon` itself is exactly `createCatalogIcon(h, CATALOG_VIEWBOX)`,
bound once per renderer. `viewBoxByName` is a plain parameter — nothing
about the factory is specific to this package's own 17-icon set. A project
with its own curated sprite gets the identical "known `name` → real
`viewBox`, no lookup at the call site, unknown `name` is a compile error"
ergonomics for its own set:

```ts
import { createElement } from 'react'
import { createCatalogIcon } from 'jsr:@zanix/space-ui@[version]'
import type { CreateElement } from 'jsr:@zanix/space-ui@[version]'

const MY_ICON_VIEWBOX = { logo: '0 0 32 32', 'chevron-down': '0 0 16 16' } as const
export type MyIconName = keyof typeof MY_ICON_VIEWBOX

export const MyIcon = createCatalogIcon(
  createElement as unknown as CreateElement<ReactElement>, // safe: the factory only ever calls h with a plain string tag + plain props object
  MY_ICON_VIEWBOX,
)
```

**When to reach for this instead of plain `Icon`**: whenever there's a *map*
of names to look up — a single icon with a `viewBox` already known at the
call site doesn't need a factory at all, `Icon` covers that directly.
**When to reach for this instead of extending `CATALOG_VIEWBOX` directly**:
you can't — `CATALOG_VIEWBOX`/`CatalogIconName` are this package's own
closed set, not extensible from outside (adding to them means forking the
package). `createCatalogIcon` is the supported way to get the identical
pattern over icons this package doesn't know about, without this package
needing to know about them either.

Same zero-cost guarantee as `CatalogIcon` itself — a consumer who never
imports `createCatalogIcon` pays nothing for its existence.

## `catalog.svg`: a template asset, not a runtime dependency

A single curated sprite at `src/templates/shared/icons/catalog.svg` — one
`<symbol id="...">` per `CatalogIconName`, each with its own real `viewBox`
(not normalized — 7 of 17 use a narrower box than the common `0 0 512
512`). Lives in this package's own `src/templates/` (published as raw
source on JSR, browsable, but **never part of the `exports` map or imported
by any `.ts` file**). A project only receives it through explicit
scaffolding — it never ships as part of installing this package as a
library dependency. Every symbol uses `fill="currentColor"`, never a
hardcoded color — an icon's visible color comes entirely from the CSS
`color` property of the `<svg>` or its nearest ancestor.

Licensed **CC BY 4.0**, sourced from `@fortawesome/fontawesome-free@7.3.1` —
full license text at `LICENSES/fontawesome-free-7.3.1.txt`, provenance in
`NOTICE.md`. These files are never renamed, minimized, or moved by anything
in this package's own build.

## Independence from theme

The catalog lives under `src/templates/shared/` — **never**
`src/templates/theme/`. Structural, not conventional: nothing inside
`catalog.svg` references a `--space-*` token, a file under `theme/`, or any
specific visual identity. A theme only ever controls *presentation* (the
`color` a symbol inherits, the `size` passed to `Icon`/`CatalogIcon`) —
never whether the catalog exists. All combinations are valid and
independent: no theme + no catalog, no theme + catalog, any theme + catalog,
any theme + no catalog. Changing or deleting a project's theme never
touches `assets/icons/`, and vice versa.

## Caution: `svgo`'s default config breaks a multi-symbol sprite

`catalog.svg` ships unminified — minification is left to the consuming app.
If a project runs it through `svgo` (directly, or via `@zanix/space`'s
`optimize.svg` build option): **`svgo`'s plain default config strips every
`<symbol id>` from a multi-symbol sprite like this one** — its `cleanupIds`
transform only ever analyzes one file at a time, so it has no way to know
an id like `catalog.svg#search` is referenced from a separate document via
`<use>`. This silently breaks every icon reference if not accounted for.
Pass the ids to keep to `svgo`'s own `preserve` option — `@zanix/space`'s
bare `optimize: { svg: true }` already does this automatically for every
`<symbol id>` while still cleaning up genuinely unused ids elsewhere in the
file, so prefer that path over a hand-rolled `svgo` config.

**`--icons` scaffolding**: a `--icons` convenience flag for `zanix new
space`/`zanix new spacecraft` that scaffolds this catalog into a project's
own `assets/icons/` and generates a pre-wired wrapper is implemented in
`@zanix/cli` (`space-icons.ts`'s `copyIconCatalog`/`writeIconCatalogFiles`/
`writeCatalogIconWrapper`, forwarded from both `newSpaceAction` and
`newSpacecraftAction`). `@zanix/space-ui` is a real, published JSR package
(published `0.1.0` onward; `ZANIX_DEPENDENCY_VERSIONS` in `@zanix/cli`'s own
`src/utils/config/dependencies.ts` carries a real `'@zanix/space-ui'` entry)
— `resolveSpaceUiVersion()`'s throw is a permanent gate against that entry
ever going missing (see that function's own doc), not a "not published yet"
placeholder — `--icons` fetches and scaffolds the real catalog today.

## Checklist before working with icons

- [ ] Is `CatalogIcon` (or a project's own `createCatalogIcon`-built
      equivalent) used for a *set* of icons — not for a single icon with an
      already-known `viewBox`, which `Icon` alone already covers?
- [ ] If optimizing `catalog.svg` (or a project's own multi-symbol sprite)
      with `svgo`, is the `preserve` option (or `@zanix/space`'s
      `optimize: { svg: true }`) actually applied — not a bare default
      config that would silently strip every symbol id?
- [ ] If citing `resolveSpaceUiVersion()`'s throw path, is it described as
      the permanent "entry ever went missing" gate it actually is — not as
      a "not published yet" placeholder (`@zanix/space-ui` is a real,
      published JSR package; `--icons` scaffolds successfully today)?
- [ ] If adding a new name to a project's own catalog, is it added to that
      project's own `viewBoxByName` map (never to this package's closed
      `CATALOG_VIEWBOX`, which isn't extensible from outside)?

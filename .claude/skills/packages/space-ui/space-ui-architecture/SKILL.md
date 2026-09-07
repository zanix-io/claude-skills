---
name: space-ui-architecture
description: The seven seams every current and future @zanix/space-ui component must keep true, and the ownership map dividing this package from the consuming Application and @zanix/space. Use before designing any new stateful component, or when reviewing whether a component change crosses into router/data-fetching/form-state territory it shouldn't own.
---

`@zanix/space-ui` is a component library, not a framework — this skill is the
boundary that keeps it that way. For the actual per-component build workflow
and real precedent bugs found while building the components, see
`space-ui-component-patterns` — it has the current component roster; don't
treat any specific count written here as current, it will drift as the
package evolves. File:line references point at
`~/Documents/Development/ZanixLibraries/space-ui` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- Check a proposed component/change against the seven seams and the
  ownership map directly — these are a fixed, closed set, not something to
  re-derive from first principles for every review.
- The rejected-abstractions list exists so a proposal resembling one of them
  doesn't need to be re-argued from scratch — cite the existing rejection
  and its reasoning instead of re-litigating it.

## The seven seams

Every current and future stateful component must keep these true. They're
not primitives (no code implements "seam 1") — they're the constraints that
keep the primitives and components composable without this package
absorbing router/data/form-state responsibility:

1. **Controlled-first, always.** Real state (`open`, `selected`, `checked`,
   `value`) is a plain prop + callback (`Modal`'s `open`/`onClose` is the
   reference shape) — never read from URL/router/storage internally. A
   component may default to uncontrolled for the simple case, but the
   controlled escape hatch must exist from the day it ships, not bolted on
   later.
2. **State survives navigation by WHERE it lives, not by a new mechanism.**
   Whether an overlay survives a client-side navigation is a question of
   whether its controlling state (and, via `@zanix/space`'s own Comets
   `persist`, its DOM) lives outside the region that gets swapped — never
   something this package itself needs to solve. `ModalProvider` works this
   way by construction: an ordinary Context Provider, safe to mount once at
   an app-shell root a navigation never touches.
3. **Live announcements are per-component, not centralized.** Each
   interactive component that needs to tell assistive tech about a
   transient change (`Slider`'s "Slide N of Total") owns its own
   visually-hidden live region, the same shape `live-region.ts`'s
   `VISUALLY_HIDDEN_STYLE` already uses. No global announcer singleton —
   extract a shared helper only once a *second* real component needs the
   identical shape, the same bar that already justified extracting
   `close-on-outside.ts`.
4. **Focus management stays factored, even before it's shared.** `Modal`'s
   capture/trap/restore logic lives in one clean effect specifically so
   lifting it out later (once a second consumer is real) is a move, not a
   rewrite. Don't let this logic entangle with anything else
   component-specific in the meantime.
5. **Prefer native HTML mechanisms for progressive enhancement.** A
   disclosure/overlay should reach for `<details>`/`popover`/`<dialog>`
   where the design allows it, before a hand-rolled JS state machine — the
   concrete failure mode this avoids (a mobile nav toggle permanently inert
   without JS) is real and documented, not hypothetical.
6. **The first render is deterministic, always.** SSR output and the first
   client paint must match without reading `window`/`location`/`matchMedia`
   during render — start from an explicit "no real measurement yet" state
   (e.g. `null`), refine after mount. Mandatory for any future component
   with client-only state.
7. **This package presents data, never owns it.** No fetch, no
   router/history calls, no form submission state, no dirty-tracking. A
   field/overlay/list component takes already-resolved data as props and
   calls back on interaction — full stop, no exceptions carved out for
   convenience.

## Ownership map

| Layer | Owns |
| --- | --- |
| **`space-ui`** | Controlled presentational components; ARIA passthrough (`aria-current`, `aria-expanded`); focus management inside a component's own subtree; interaction mechanisms once genuine repetition exists |
| **Application** | Which state is "current" (which link, which tab); reading the URL and passing the derived value down as a controlled prop; deciding where to mount a Provider component; data-fetching, optimistic policy, dirty-tracking |
| **`@zanix/space`** | Routing/navigation (Orbit); hydration timing and cross-navigation state/DOM persistence (Comets, `persist`); server-driven form progressive enhancement (`loader`/`action`/`fieldErrors`/`submitted`, 422 re-render) — deliberately with no client form-state layer |
| **Never `space-ui`, permanently** | Data-fetching of any kind; router/history manipulation; form submission/dirty-tracking state; URL parsing |

If a proposed component or prop would need this package to fetch, read the
URL, call router/history, or track form dirty state — it belongs to the
Application or `@zanix/space`, not here, regardless of how convenient it
would be to add.

## Deliberately rejected abstractions

- **A generic `Presence`/`Show`/`When`/`Toggle` wrapper** — the plain
  `{condition && <Panel/>}` idiom already covers every real case, confirmed
  byte-for-byte equivalent by test; no named abstraction exists anywhere in
  the codebase this descends from despite the pattern recurring 3+ times
  unfactored. That recurrence is evidence *against* even a private helper,
  not for one.
- **A fused "dismissable layer"** (outside-click + `Escape` + focus-trap as
  one primitive) — the combinations genuinely diverge (`Toast` needs none of
  the three, `Combobox` likely wants `Escape` without a full trap, only
  `Modal`/`Drawer` want all three together). Kept as three separate,
  composable pieces instead (see `space-ui-foundation-primitives`).
- **A generic keyboard-interaction "key → handler" map** — adds no value
  beyond direct `onKeyDown` composition; the one real transversal sub-need
  (roving focus) is already named as its own primitive.
- **Any form-state/submission-state machinery** — `@zanix/space` already
  decided against this architecturally; this package's only legitimate
  opening is a presentational field component consuming already-resolved
  error messages as plain props, with zero state of its own.
- **URL-reading, router/history calls, or data-fetching inside any
  component** — always external, no exceptions, regardless of how many
  future components would find it convenient.

Before proposing one of these (or something shaped like it), check this list
first — a fresh argument for a `Presence` component or a fused dismissable
layer needs to overcome the specific, already-recorded evidence against it,
not just seem reasonable in isolation.

## Export surface: the root barrel vs. `./runtime/<name>` — a real cross-package dependency

Distinct from the ownership map above (which governs what a component may
*do* at runtime — fetch, route, own form state); this governs where a
component's own module gets *exported from*, based on what it actually
*imports*.

`mod.ts`/`mod-preact.ts` (the package's `.`/`./preact` entrypoints) export
the majority of this package's components and have **zero** runtime
dependency on `@zanix/space` — confirmed by a permanent, table-driven
structural test (`src/@tests/unit/intl/dependency-boundary.test.ts`, walking
`deno info --json`'s real resolved module graph, one row per component).
`Video`, `Image`, `RichText`, `ImgButton`, `Card`, and `NavDrawer` are the
real exception: `Video`/`Image` resolve `@zanix/space/assets-manifest`'s
own `resolveAssetHref` directly; `RichText` resolves it directly too, and
also composes `Image`'s, `Video`'s, and `ImgButton`'s own `render.ts`
factories internally for its built-in tags; `ImgButton` and `Card` each
compose `Image`'s `render.ts`; `NavDrawer` reaches `@zanix/space` a
different way — it's a real `'use comet'` Comet, importing `defineComet`
from `@zanix/space/comet` directly, never `resolveAssetHref`. **`Menu` is
NOT in this list** — despite an earlier version of this skill naming it
here, `Menu` was changed to compose only `Link`/`Button`/`Icon` (via its
own `visual` render-prop, never `Image`/`ImgButton` directly) specifically
*to* remove its `@zanix/space` dependency and make it usable inside a
`'use comet'` file; it now has zero such dependency and lives in the root
barrel with the rest (see `components/Menu/index.ts`'s own doc for the full
story, and the CHANGELOG entry that made this change).

Each of the six real exceptions is exported from its OWN single-component
subpath (`./runtime/video`, `./runtime/image`, `./runtime/rich-text`,
`./runtime/img-button`, `./runtime/card`, `./runtime/nav-drawer` — a
`/preact` variant alongside each, e.g. `./runtime/image/preact`) — never
from the root barrel, and **never from a shared combined `./runtime`
barrel either**. A combined `./runtime`/`./runtime/preact` entrypoint DID
exist for a while (all six re-exported from one `src/runtime.ts` file) —
removed once it produced the identical bug described below, one level in
(see "Why per-component subpaths, not just a separate entrypoint" below).

**Why a separate entrypoint at all, not a style preference**: a barrel
export forces resolution of everything it re-exports together. With these
six inside `mod.ts`/`mod-preact.ts`, importing even ONE unrelated component
from there (e.g. `Button`, which has zero `@zanix/space` dependency) would
force resolution of these six as well — pulling `@zanix/space` back into
the graph. Since `@zanix/space`'s own build pipeline is what resolves a
`@zanix/space-ui` import when building a `@zanix/space` app that uses this
package, that produced a genuine circular resolution (`@zanix/space`'s own
build tooling needing to resolve `@zanix/space` itself, one repo away) —
confirmed to hang `@deno/loader`'s native workspace resolution in a real
`zanix space build`. Splitting these six out of `.`/`./preact` fixed that.

**Why per-component subpaths, not just a separate combined entrypoint**:
the same barrel-forces-resolution mechanism recurs at ANY shared barrel,
not just the root one — confirmed real when `NavDrawer` was added to the
shared `./runtime` entrypoint alongside the other five: `import { NavDrawer
} from '@zanix/space-ui/runtime'` resolved the one shared `runtime.ts`
file, which ALSO statically re-exported `RichText` (and `Video`/`Image`/
`ImgButton`/`Card`) in that same file — pulling `RichText`'s own
`markdown-to-jsx`/`@zanix/helpers` chain into `NavDrawer`'s real build
graph, even though `NavDrawer`'s own module never imports any of them
(confirmed via `deno info --json` on the shared file, before the fix — all
of `RichText`'s own module files were present). Splitting into one subpath
per component fixed this the same way splitting out of the root barrel did
the first time: importing one `./runtime/<name>` subpath never statically
or dynamically reaches another's own files, unless it genuinely composes it
(`ImgButton`/`Card` composing `Image`'s `render.ts`; `RichText` composing
`Image`'s, `Video`'s, and `ImgButton`'s — all real, reviewed, intentional
composition, not a barrel accident).

**The rule for any FUTURE component**: trace its own real reachable
graph — its own direct imports, AND any component it composes internally
via a relative import (build the full transitive closure, not just one
level deep). If that graph touches `@zanix/space` (or any other real
cross-package runtime dependency) anywhere, give it its OWN NEW
`./runtime/<kebab-name>`/`./runtime/<kebab-name>/preact` subpath — never
the root barrel, and never an existing component's own `./runtime/*` file
or any shared combined barrel, even a narrow one covering just two or three
components (that's exactly the bug class that already recurred once).
A component with zero such dependency stays in the root barrel with the
rest — `README.md`'s "Current status" section is the live count of which
components that is, not a number restated here (see this skill's own header
note on why). `src/@tests/unit/intl/dependency-boundary.test.ts`'s own
table-driven suite is the permanent regression guard for this whole
section — a new component means one new row, in the appropriate table,
with its own exact expected footprint; it isn't optional bookkeeping.

- [ ] Does this component's own module import `@zanix/space` (or any other
      real cross-package runtime dependency) directly?
- [ ] Does it compose another component internally (a relative import into
      that component's own `render.ts`/`index.ts`, not just a shared type)?
      If so, does THAT component (transitively, at any depth) reach
      `@zanix/space`?
- [ ] If either is true, does the new component get its OWN new
      `./runtime/<kebab-name>` (+ `/preact`) subpath — never an existing
      component's `./runtime/*` file, never a shared combined barrel, never
      `.`/`./preact`?
- [ ] Has a new row been added to `dependency-boundary.test.ts`'s own table
      (root-barrel or `./runtime/*`, whichever applies) with an exact,
      reviewed expected footprint?

## A second cross-package hazard: a server-capable import inside browser-bundled code

Distinct from the `@zanix/space` export-surface hazard above (which is about
where a component's exports LIVE); this one is about which ENTRY of a
dependency a component's own module imports, and it hit `Modal`/`Drawer`
specifically: both originally imported `@zanix/utils/logger` (the full,
server-capable entry — `WorkerManager`, `Deno.readTextFile`-backed default
storage) directly for one `logger.warn` call. Neither resolves to a local
file for a browser bundler, so bundling either component pulled in real,
remote `https://jsr.io/...` fetches for `@std/fmt/colors`/`@std/path` on
every page load — a confirmed, reproduced regression, fixed by routing both
through the new `src/shared/client-logger.ts` (wraps `@zanix/utils/logger/
client`'s browser-safe `createClientLogger`; see that module's own doc for
the full mechanism and reasoning) instead of `@zanix/utils/logger` directly.
`@zanix/space`'s own `modules/client/client-logger.ts` established the
identical pattern for the identical reason first.

**The rule for any FUTURE component that needs to log anything**: import
`shared/client-logger.ts`, never `@zanix/utils/logger` directly — this
applies regardless of whether the component itself is in the root barrel or
any `./runtime/*` subpath, since the hazard is about what a *browser bundler* resolves,
not about `@zanix/space` reachability (the concern the export-surface rule
above governs). Any dependency with a split "full" vs. "client-safe" entry
(check its own `deno.json(c)` `exports` map for a `/client` subpath before
assuming only one entry exists) needs the same treatment the first time a
component reaches for it from browser-bundled code.

## Checklist before designing a new stateful component

- [ ] Is all real state a controlled prop + callback, with an uncontrolled
      default available but never required?
- [ ] Does anything here read `window`/`location`/`matchMedia` during
      render, risking an SSR/first-paint mismatch?
- [ ] Does a live announcement (if any) live inside this component's own
      subtree, not a shared/global announcer?
- [ ] Does this component fetch, read the URL, call router/history, or
      track dirty state anywhere? If yes, it doesn't belong in this
      package.
- [ ] Does the design resemble a previously-rejected abstraction (`Presence`,
      a fused dismissable layer, a generic key-handler map)? If so, does it
      have a genuinely new argument, or does the original rejection still
      hold?
- [ ] Does this component log anything? If so, does it import
      `shared/client-logger.ts`, never `@zanix/utils/logger` directly — see
      this skill's own "A second cross-package hazard" section above?

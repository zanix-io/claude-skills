---
name: space-ui-styling
description: @zanix/space-ui's headless styling architecture — className as the only styling prop, data-space-ui as a stable (not styling) selector hook, the theme/ vs shared/ starter templates, --space-* token composition, why BEM/Tachyons aren't part of this package, and the nonce prop Modal/Drawer/Toast/Tooltip/Popover accept to stay CSP-compliant (a component-rendered `<style nonce>` element plus, for Tooltip/Popover's dynamic offset, CSSOM rule mutation — never an inline style attribute or an external stylesheet). Use when adding data-space-ui to a new component, writing/reviewing an optional stylesheet against this package's components, deciding whether new CSS belongs in theme/ or shared/, or wiring a component's functional positioning under a strict CSP.
---

File:line references point at `~/Documents/Development/ZanixLibraries/space-ui`
— read the real code there before assuming this summary is still accurate.

## Golden rule (token savings)

- Confirm a specific component's `data-space-ui` value(s) against the real
  table in `docs/styling.md` directly rather than reading that component's
  own source — the table is kept in sync with real markup.
- Deciding `theme/` vs `shared/` for new CSS is a one-question check ("does
  this vary by visual identity, or is it structural/behavioral regardless of
  theme?") — apply it directly.

## Headless by default — the permanent architecture, not a gap

No component imports a stylesheet, generates a class name, or assumes any
styling tool is present. `className?: string` is the one styling prop every
component accepts, forwarded verbatim to its root element — **never merged
with, or replaced by, anything this package authors itself.** This isn't
temporary; it's the permanent architecture. A consuming app styles every
component with whatever it already uses (Tailwind, CSS Modules,
vanilla-extract, plain CSS, or nothing).

## `data-space-ui`: a stable identity hook, not a styling system

Every component renders (or inherits, when composing another — see
`space-ui-component-patterns`) a `data-space-ui="<name>"` attribute — inert
on its own, nothing in this package reads or reacts to it. It exists so an
*optional* stylesheet can select against a stable identity without a bare
element selector (`button`, `svg`, `a`) that would also match unrelated
markup elsewhere on the page.

**It's a stable, semver-protected contract** — not because arbitrary
external CSS is expected to depend on it (`className` remains the primary
supported styling path), but because this package's own official scaffolded
template does: renaming or removing a `data-space-ui` value is a breaking
change to this package, exactly like removing a documented prop would be.
Adding a new `data-space-ui` value for a new component, or changing an
existing one, needs the same care as a public API change — see
`space-ui-component-patterns`'s checklist for when a new component composes
another and should inherit its hook instead of adding a new one.

## `theme/` vs `shared/` — two independent optional starter templates

Scaffolded (once wired) by `zanix new space`, living under this package's own
`src/templates/`, never imported by any runtime code here. A project that
receives either owns the file outright — no version pin, no import, edit or
delete freely. They answer different questions, and a project can pick
either, both, or neither:

- **`theme/tokens.css`** — "what does this look like." A starter default
  palette (primitive scale values + semantic tokens referencing them),
  following `@zanix/space`'s own primitive/semantic convention exactly. The
  one file that varies by visual identity — a different theme preset would
  ship a different `tokens.css`, same shape.
- **`theme/space-defaults.css`** — the one file, of the four, that styles
  `@zanix/space`'s OWN markup contract (`data-space`, never this package's
  own `data-space-ui`): its built-in `not-found`/`error` fallback views, and
  every `@zanix/cli`-scaffolded template's root element, all sharing ONE
  generic `[data-space="content"]` hook (never a per-`--template` value) so a
  future scaffolded template inherits it with zero changes here. References
  only semantic tokens from `theme/tokens.css`, same discipline as
  `shared/behavior.css`. **This is the canonical source** — `@zanix/cli`'s
  own `space-theme.ts` ships an embedded, byte-for-byte-synced copy (see
  `docs/styling.md` for the integrity test), rather than fetching it from
  JSR at scaffold time.
- **`shared/behavior.css`** — "what does this generic interaction pattern
  need," regardless of theme. Structural/animation CSS, deliberately
  theme-agnostic (only references semantic token *names*, never a literal),
  keyed off `data-space-ui` attribute selectors — never a class name a
  component would have to render itself. Includes a
  `prefers-reduced-motion: reduce` override where an animation exists
  (`ProgressBar`).
- **`shared/card.css`** — kept as its own file, separate from
  `behavior.css`, because `Card`'s responsive layout is CSS *layout*, not
  the animation/structural-behavior concern `behavior.css` covers.

**A component's functional correctness never depends on either template
being loaded** — e.g. `Modal`/`Drawer`'s stacking (`z-index`) stays owned by
each component itself, deliberately not CSS-dependent, so headless
correctness holds even with zero CSS ever imported. Don't move a functional
concern like stacking order into an optional stylesheet just because a token
exists for it — that would make headless behavior depend on an optional
file, which is a regression, not a simplification.

## Functional positioning under a strict CSP: `nonce`

`Modal`, `Drawer`, `Toast` (via `ToastProvider`), `Tooltip`, and `Popover`
all need real `position`/`z-index` (plus a per-instance anchor) to function
as an overlay at all — the same "functional, not decorative" exception
above, not a headless regression. Applying that as an inline `style`
attribute is a real, confirmed-in-browser violation of a nonce-based
`style-src` CSP — `@zanix/space`'s own zero-config default is exactly this
shape (`space-middleware-and-security`) — since a CSP nonce never applies to
a `style="..."` attribute, only to a `<style>` element/`<link
rel=stylesheet>`, and browsers block `element.style.setProperty(...)`/
`.style.cssText = ...` under the same rule too, so CSSOM-mutating an
element's own inline style doesn't sidestep it either.

All five components instead render their own `<style nonce={nonce}>`
element, built once at module scope from the same style-object constants
they always used (`MODAL_POSITION_STYLE`/`MODAL_Z_INDEX`/
`DRAWER_SIDE_STYLE`/`DRAWER_Z_INDEX`/…), keyed off
`data-space-ui`/`data-position`/`data-side` attribute selectors instead of
an inline attribute — this keeps "zero CSS import required" intact (the
rule ships with the component, not an external stylesheet the consumer has
to remember to import) while becoming CSP-compliant once a nonce is
supplied. See `shared/overlay-position-css.ts` (`buildOverlayCss`) for the
shared CSS-building helper, and each component's own `nonce?: string` prop
doc for the per-component contract. A consumer under no strict CSP passes
nothing — the `<style>` tag still applies exactly as before; a page under a
strict nonce-based `style-src` (like `@zanix/space`'s own default) must
thread its real per-request nonce down as this prop, or these five
components' positioning silently fails to apply. There is no
external-stylesheet alternative offered for this — moving it there would
require every consumer to import an extra file just to get a working
overlay, breaking headless-by-default for everyone, not just strict-CSP
consumers.

**`Tooltip`/`Popover`'s own genuinely dynamic positioning is covered too**,
not just the static anchor: their panel's real offset — a `transform:
translate(x, y)` (plus `visibility`/`pointer-events`) recomputed every
render from a live `usePosition` measurement — can't be expressed as a
static rule the way a fixed enum-keyed constant can, so it doesn't use
`buildOverlayCss`. Instead it's applied to a CSSOM rule scoped to that one
component instance (`[data-space-ui='tooltip'][data-tooltip-id='...']`),
inserted once via `sheet.insertRule(...)` into the SAME `<style
nonce={nonce}>` element already rendering the static rule, then mutated on
every position update via `CSSStyleRule.style.setProperty(...)` — never
`HTMLElement.style`, which is what `style-src-attr` actually covers. A CSP
nonce authorizes the `<style>` ELEMENT itself once; CSSOM mutation of a rule
already living inside that authorized element is a distinct code path from
mutating an inline `style` attribute — the same technique CSP-compatible
CSS-in-JS runtimes (styled-components' "speedy" mode, Emotion) use. Applies
`useLayoutEffect` (not `useEffect`) so the mutation lands before paint, same
as the synchronous inline-style update it replaces — a plain `useEffect`
would cause a visible flicker/jump on every scroll-triggered position
update, since `autoUpdate` re-measures continuously while open, not just on
mount.

Don't reach for `CatalogIcon`/any asset-backed styling mechanism to solve a
problem like this one — the fix stays entirely self-contained (a
component-rendered `<style>` element plus CSSOM), never an external asset
the consumer must supply an `href` for. That would reintroduce the same
"depends on an optional file" regression this section opened with.

**A real, cosmetic-only React hydration warning this introduces, and why the
fix lives outside `render.ts`.** A browser clears an applied `nonce`
CONTENT ATTRIBUTE back to `""` right after using it (spec'd behavior — the
real value survives only on the element's own `.nonce` property). React's
hydration mismatch check special-cases this for `<script>` (reads `.nonce`
instead of the attribute) but not for `<style>`, so a server-rendered
`<style nonce="real-value">` logs "A tree hydrated but some attributes of
the server rendered HTML didn't match..." on every page load under a
nonce-based CSP — confirmed live against a real `@zanix/space` page;
functionally harmless (the nonce still applies, CSP still passes), just
noisy. The fix, `suppressHydrationWarning: true`, is a REACT-ONLY convention
— Preact's `h` has no equivalent special case (an unknown prop name falls
through to a literal `setAttribute`), so adding it directly in the shared
`render.ts` would leak a real `suppresshydrationwarning="true"` attribute
into Preact's own rendered/SSR markup. Each of the five components' React
binding (`index.ts`, never `index.preact.ts`) instead passes
`shared/create-element-nonce-hydration-fix.ts`'s
`createElementWithNonceHydrationFix` in place of raw `createElement` — it
adds `suppressHydrationWarning` only to a `<style>` element that actually
carries a `nonce` prop key, leaving everything else (and every Preact
binding) untouched. If a future component needs its own `<style nonce>`
element, reuse this same wrapper in its `index.ts` rather than reinventing
the fix or adding `suppressHydrationWarning` to `render.ts` directly.

## `--space-*` tokens: compose, never invent a naming scheme

Any custom property this package's own templates declare follows
`@zanix/space`'s own primitive/semantic convention exactly — primitives are
raw scale values never referenced directly by a component; semantics are
named roles that reference a primitive, the only level a component or
`shared/behavior.css` may consume. **This package never invents its own
token naming scheme** — it composes with whatever `--space-*` tokens the
host app's `globalCss`/`theme.resolve` already resolve, via the normal CSS
cascade.

## BEM and Tachyons: resolved, not carried forward

BEM is not a styling/collision-avoidance mechanism of this package — there's
no runtime `useStyles`-style hook, and no component depends on a specific
class name to function. BEM naming (`block__element--modifier`) is still
available as an optional preference inside a project's own CSS Module, but
never something this package imposes, documents as official, or validates.
Tachyons is not part of this package's implementation or contract either —
its role (atomic utility CSS) is already covered by Tailwind v4,
`@zanix/space`'s own default. Neither is prohibited in a consuming app's own
code; what changed is that this package itself doesn't ship, require, or
validate either.

## Replacing styling completely

Because every component's only styling surface is `className` plus the
inert `data-space-ui` hook: don't scaffold `theme/`/`shared/` (or delete
them if already scaffolded), and style `[data-space-ui="..."]` — or just
pass `className` per instance — from whatever CSS system the app already
uses. Nothing in this package's own runtime needs to know about or agree
with that choice.

## Checklist before adding/reviewing styling-related work

- [ ] Does the new/changed component forward `className` verbatim, never
      merging or replacing it with anything authored internally?
- [ ] Does a new component's `data-space-ui` value inherit from a composed
      component where one exists, per `space-ui-component-patterns`, rather
      than adding a redundant one?
- [ ] Is new CSS going into `theme/` (varies by visual identity) or
      `shared/` (structural/behavioral, theme-agnostic) — the right one for
      what it actually does, not by convenience?
- [ ] Does any *functional* correctness (stacking, positioning) accidentally
      end up depending on an optional stylesheet being loaded? It shouldn't
      — that's a regression from headless-by-default.
- [ ] Does new CSS reference only semantic `--space-*` tokens (never a
      primitive directly, never a literal value), and never invent a new
      token naming scheme of its own?
- [ ] If a new/changed component needs functional `position`/`z-index` (or
      any other genuinely functional, non-decorative style) to work at all,
      does it render its own `<style nonce={nonce}>` element instead of an
      inline `style` attribute — never an inline attribute (blocked by a
      nonce-based CSP with no workaround), and never an external stylesheet
      the consumer must import (breaks headless-by-default for everyone, not
      just strict-CSP consumers)? If any part of that style is genuinely
      dynamic (recomputed every render), does it use CSSOM rule mutation
      inside that same `<style>` element via `useLayoutEffect`, not
      `HTMLElement.style`?
- [ ] Does a new `<style nonce={nonce}>` element's React binding pass
      `createElementWithNonceHydrationFix`
      (`shared/create-element-nonce-hydration-fix.ts`) instead of raw
      `createElement`, to avoid a real (cosmetic-only) hydration-mismatch
      warning on every page load — and does the Preact binding stay on raw
      `h`, never `suppressHydrationWarning`, which Preact doesn't recognize?

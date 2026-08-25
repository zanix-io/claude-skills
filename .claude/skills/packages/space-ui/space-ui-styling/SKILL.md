---
name: space-ui-styling
description: @zanix/space-ui's headless styling architecture — className as the only styling prop, data-space-ui as a stable (not styling) selector hook, the theme/ vs shared/ starter templates, --space-* token composition, and why BEM/Tachyons aren't part of this package. Use when adding data-space-ui to a new component, writing/reviewing an optional stylesheet against this package's components, or deciding whether new CSS belongs in theme/ or shared/.
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
each component's own inline `style`, deliberately not CSS-dependent, so
headless correctness holds even with zero CSS ever imported. Don't move a
functional concern like stacking order into an optional stylesheet just
because a token exists for it — that would make headless behavior depend on
an optional file, which is a regression, not a simplification.

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

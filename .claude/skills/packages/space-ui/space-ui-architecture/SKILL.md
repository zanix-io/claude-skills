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

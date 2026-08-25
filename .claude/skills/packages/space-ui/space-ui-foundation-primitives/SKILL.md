---
name: space-ui-foundation-primitives
description: The shared/ toolkit @zanix/space-ui components are built on — useCloseOnOutside, focus-scope, live-region, roving-focus, positioning/usePosition, the overlay stack — what's public vs. internal, and the extraction criterion (a real second consumer, not speculation) that governs both. Use when building a component that needs one of these behaviors, or deciding whether new shared logic should become a public primitive.
---

For the architectural constraints these primitives exist to serve, see
`space-ui-architecture`. For how a component actually gets built using them,
including real bugs found along the way, see `space-ui-component-patterns`.
File:line references point at `~/Documents/Development/ZanixLibraries/space-ui`
— read the real code there before assuming this summary is still accurate.

## Golden rule (token savings)

- Check this skill's table for whether a primitive already exists before
  building new shared logic — most interaction needs (outside-click,
  Escape-to-close, focus trapping, live announcements, roving tabindex,
  floating-panel positioning) already have one.
- The extraction criterion (below) is a one-question check — "is there a
  real second consumer?" — apply it directly rather than re-deriving
  whether something should be shared.

## The extraction criterion: a real second consumer, never speculation

Nothing here was extracted ahead of a real second use. `focus-scope.ts` was
pulled out of `Modal`'s own inline logic and made public specifically because
`Drawer` was coming — but the criterion was still a **known, real** second
consumer, not a hypothetical future one. Don't extract shared logic from a
single component "in case it's needed later" — wait for the second real
consumer, the same bar every primitive below was held to.

## Foundation primitives

| Primitive | Status | Real consumers |
| --- | --- | --- |
| `close-on-outside.ts`/`.preact.ts` (`useCloseOnOutside`) | Public | `Modal`, `Menu` ×2 |
| `escape-to-close.ts` (`createEscapeToCloseHandler`) | Public | `Menu`'s toggle + each submenu item (2, byte-identical) — **not** `Modal`, whose own `Escape` handling is merged with `Tab`-cycling and doesn't do inline refocus; forcing it onto this shape would cost correctness for no real simplification |
| `shared/overlay-stack.ts` (`registerOverlay`/`isTopOverlay`) | Internal (not exported from `mod.ts`/`mod-preact.ts`) | `Modal`, `Drawer` — genuinely ONE shared stack, so a `Modal` and a `Drawer` open at once correctly defer to whichever is truly topmost, regardless of kind |
| `focus-scope.ts`/`.preact.ts` (`useFocusScope`: capture → trap → restore) | Public | `Modal` (1 real consumer today — extracted ahead of `Drawer` per explicit decision not to gate foundation primitives on current consumer count once a near-term second one is known) |
| `live-region.ts` (`liveRegionProps`, `VISUALLY_HIDDEN_STYLE`) | Public | `Slider` (1 real consumer — `Toast` composes `Alert` instead, since a toast is VISIBLE and this is for announcement-only regions, so it stays at 1) |
| `roving-focus.ts` (`getNextRovingIndex`, `createRovingKeyDownHandler`) | Public | `RadioGroup`/`Tabs` (via `createRovingKeyDownHandler`, real focus moves) and `Combobox` (via `getNextRovingIndex` directly, `aria-activedescendant` — focus never leaves the input) — the split into two functions was anticipated for exactly this reason |
| `positioning.ts` (`computePosition` — placement/offset/flip/shift, pure geometry) | Public, full engine (12 placements, collision detection, flip, shift) | `Popover`/`Tooltip`/`Combobox` (via `usePosition`) — built as the full engine ahead of its first consumers, a deliberate choice over a minimal version |
| `positioning-dom.ts` (`measurePosition`, `autoUpdate`) | Public | Real-element measurement (`getBoundingClientRect`/viewport/scroll-parent) and `ResizeObserver`+scroll/resize live updates |
| `use-position.ts`/`.preact.ts` (`usePosition`) | Public | `Popover`/`Tooltip`/`Combobox` — SSR-safe (`null` until first client measurement, ref-gated) |
| Public `announce(message)` (app-level, component-independent) | Named, not adopted | No scenario has identified a real app-level (not component-owned) announcement need |

## Two sharp edges worth knowing before touching these

- **`usePosition` tracks `active` + `JSON.stringify(options)`, never the raw
  `options` object.** Don't "simplify" this back to the raw object — see
  `space-ui-component-patterns`'s "real bugs already found and fixed" for
  why (a real, confirmed infinite-render-loop bug), and for the general
  lesson it generalizes to for any new effect.
- **`escape-to-close.ts` isn't a universal `Escape` handler** — it's
  specifically shaped for `Menu`'s two real consumers. `Modal`'s own
  `Escape` handling is deliberately separate because it's merged with
  `Tab`-cycling in a way this primitive doesn't cover. Don't force a new
  component's `Escape` handling onto this shape just because it exists;
  confirm the shape actually matches before reusing it.
- **`Button` is a plain function component, never `React.forwardRef`-wrapped
  — no new component composing it can get a real DOM ref to it directly.**
  This hits any component that composes `Button` as a trigger while also
  needing a real element for `usePosition`/an explicit refocus target:
  the fix is always to wrap `Button` in an owned element (e.g. a `<span>`)
  and query it fresh from that wrapper's ref, never to expect a ref prop to
  reach the button itself — the same technique `Popover`'s `referenceRef`,
  `Menu`'s `toggleWrapperRef`, and `Select`'s own trigger wrapper all
  independently arrived at. Check `Button/types.ts`'s real closed prop
  surface before assuming a composed trigger can take an arbitrary prop
  too — no `onKeyDown`, `aria-describedby`, `aria-invalid`, or
  `aria-haspopup` today; extending it is a separate, wider change, not
  something to route around locally.

## Public since a recent, deliberate policy reversal

These primitives are public (exported from `mod.ts`/`mod-preact.ts`, not
just internal to this package) as a deliberate policy: they're small,
self-contained, dependency-free building blocks (no `@zanix/space` coupling,
no app-state assumptions) that a consumer app builds the identical shape of
component around all the time (`useCloseOnOutside` for a custom dropdown,
`computePosition` for a hand-rolled tooltip) — withholding them would force
exactly the duplication this package exists to avoid. The renderer-split
discipline this requires internally applies to the public surface too:
`mod.ts` exports the React implementation of stateful primitives
(`useCloseOnOutside`, `useFocusScope`), `mod-preact.ts` the independent
Preact one, and both re-export the identical module for the
renderer-agnostic ones (`escape-to-close.ts`, `live-region.ts`,
`roving-focus.ts`, `positioning.ts`, `positioning-dom.ts`) — verified via the
same `deno info`-based module-graph technique `dependency-boundary.test.ts`
uses for the rest of each entrypoint's surface.

## Checklist before adding new shared logic

- [ ] Does a real second consumer already exist, or is this still
      speculative? If speculative, keep it inline in the one component that
      needs it.
- [ ] Does an existing primitive already cover this need (check the table
      above) before writing new shared logic?
- [ ] If making something public, is it genuinely dependency-free (no
      `@zanix/space` coupling, no app-state assumption)? If not, it likely
      belongs to the Application layer instead (see `space-ui-architecture`'s
      ownership map).
- [ ] If it's stateful/effect-based, does it need a separate React/Preact
      implementation, or is it pure enough to share verbatim across both?

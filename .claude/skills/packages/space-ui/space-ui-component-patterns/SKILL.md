---
name: space-ui-component-patterns
description: The discipline for building a new @zanix/space-ui component — data-space-ui hooks, controlled/uncontrolled shape, composed vs. reimplemented markup, the render-prop pattern for arbitrary caller content, and real React/Preact divergence bugs already found and fixed. Use when adding a new component, or reviewing one for consistency with the rest of the package.
---

For the architectural constraints every component must satisfy, see
`space-ui-architecture`. For the shared toolkit these patterns build on, see
`space-ui-foundation-primitives`. File:line references point at
`~/Documents/Development/ZanixLibraries/space-ui` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- Copy the shape from the closest existing sibling component (a stateless
  visual component → `Icon`/`Alert`/`Skeleton`'s `render.ts` factory
  pattern; a controlled interactive one → `Modal`/`Drawer`'s shared
  primitives; a data-driven list → `Menu`/`Accordion`/`RadioGroup`/`Tabs`'
  `items[]` shape) rather than re-deriving conventions from this skill's
  prose for every new component. **A single closest sibling isn't
  guaranteed** — a component can genuinely sit between two established
  patterns and need a hybrid (`Select` borrows `Combobox`'s structure —
  owned trigger, `usePosition`-positioned popup, `useCloseOnOutside`
  scoped to both — but `RadioGroup`/`Tabs`' automatic-activation selection
  semantics, since nothing in `Select` types, unlike `Combobox`'s own
  manual Enter-to-commit). Reading two candidates in full before deciding
  is the right move in that case, not a sign the wrong guide was picked.
- The real bugs section exists so a new component doesn't re-introduce a
  mistake already made and fixed once — check it before writing effect/event
  code that resembles one of these shapes, not as a mandatory read for
  every change.

## Three implementation shapes, not a binary — chosen by what actually differs between renderers

Earlier guidance here presented a false binary ("no hooks → shared
`render.ts`, any hooks → a full second implementation"). Corrected, after
empirically verifying the middle case (a throwaway `Counter` reimplementation,
mounted through both `react-dom`'s real `createRoot` and Preact's real
`render()`, driven by this package's own fake-clock `IntersectionObserver`/
`requestAnimationFrame` mocks — state updates re-rendered correctly, effect
cleanup fired on both unmount and a mid-animation prop change, no "invalid
hook call" in either renderer) and then applying it for real to `Table`:

- **Stateless/presentational** (no per-renderer hook usage): write the real
  logic once against `React.createElement`/`Preact.h`'s shared call
  signature — never JSX, never a runtime "which renderer am I" check — then
  bind it to each renderer exactly once. `Icon`, `Alert`, `Skeleton`,
  `VisuallyHidden` all use this `render.ts` factory pattern.
- **Stateful, with a genuinely shareable body**: the SAME `render.ts`-factory
  technique, extended to inject the hooks themselves (`useState`/`useEffect`/
  `useRef`/`useContext`/...) as a second parameter alongside `h`, not
  just `h` alone. This is sound, not just "looks the same": React's and
  Preact's hooks dispatchers key a hook's state on ITS OWN component
  instance's call order across renders, never on how the function that calls
  it was constructed — a `useState` call reached through `hooks.useState(...)`
  inside a factory-returned closure is call-order-identical, every render, to
  one written as a bare `useState(...)` in a hand-written function body.
  **Never the renderer's own bare `useId` for a value that needs to match
  between server render and client hydration**, though — see
  `space-ui-builder`'s own step 6 for the full "why" (a Comet's isolated
  hydration root) and the two correct alternatives (`@zanix/space`'s
  `useCometStableId`, or a props-derived hash for a component that must stay
  `@zanix/space`-dependency-free). Use
  this whenever a stateful component's actual body — every hook call, every
  branch, every returned element — is otherwise IDENTICAL between the two
  renderers, and differs only in which module a hook/`h` is imported from and
  the trailing `as VNode`/return-type cast. `Table`'s own `render.ts`
  (`createTable(h, hooks)`) is the reference example for this shape,
  alongside `CatalogIcon`'s plain (`h`-only) one — read both before applying
  this to a new component. A composed component (`Button`/`Link`/...) is
  still bound the same way `ImgButton/render.ts` already does (`createButton(h)`
  called once inside the outer factory, before returning the actual
  component function) — no extra injected parameter needed for those.
- **Stateful, with real renderer-specific divergence in the body itself**:
  still needs a genuine second implementation (or, if the divergence is
  small and isolable, a shared body with a narrow per-renderer branch) —
  `Counter`'s own `useEffect`/`useRef` pair was the ORIGINAL reason this
  package assumed the binary above; empirically it turned out to be sound to
  share too (see the verification above) — the real, CONFIRMED blocker case
  is `Combobox`'s own `onChange` vs `onInput`: React remaps `onChange` to
  fire on every keystroke while Preact's `onChange` means the literal native
  `change` event (fires only on blur), a real, confirmed behavioral
  divergence (not just which module something is imported from), forcing a
  different prop key (`onChange` vs `onInput`) AND a different handler body
  (`event.target.value` vs `(event.target as HTMLInputElement).value`, since
  Preact's `JSX.TargetedEvent` doesn't narrow `target` the same way). Before
  reaching for a full second implementation on this basis alone, check
  whether the divergence is actually this narrow (an isolable prop-name/
  handler swap, potentially still shareable with a small per-renderer branch)
  or genuinely pervasive enough to make a shared body incorrect — don't
  assume every stateful component needs this shape just because one hook is
  involved somewhere.

`Menu` has a smaller version of the same kind of thing worth knowing about:
its React binding wraps one branch in `Fragment` specifically to silence a
React-only dev-mode "missing key" console warning; Preact never warns for
that shape, so its own binding skips the wrapper. Harmless either way — a
shared body could apply the `Fragment` wrapper unconditionally (Preact just
ignores the now-redundant wrapping) rather than needing a real branch — but
it's a fair example of the "looks like it needs a second implementation,
turns out to be a trivial, resolvable difference once you check" pattern
`Combobox` is the opposite (real, unresolvable-without-a-branch) example of.

## Controlled-first, with an uncontrolled fallback

Every stateful component's reference shape: `open`/`onOpenChange` (or the
component's own equivalent pair) as the controlled API, with an uncontrolled
`defaultOpen`-style fallback for the simple case — see `space-ui-architecture`'s
seam 1 for why the controlled form must exist from day one, not bolted on
later.

Data-driven list components (`Menu.items`, `Accordion`'s `items:
AccordionItem[]`, `RadioGroup.items`, `Tabs.items`) all share the same
`items[]`-array shape rather than each inventing its own — reuse that shape
for a new data-driven component rather than a bespoke one.

## `data-space-ui="<name>"`: composed markup inherits, never reimplements

A component built from another real component (not just visually similar,
but actually rendering it) inherits that component's own `data-space-ui`
hook — it never adds a second, redundant one:

- `CatalogIcon` delegates to `Icon` verbatim → inherits `"icon"`, nothing of
  its own.
- `ImgButton`'s root **is** a real `Link` or `Button` → inherits
  `"link"`/`"button"` accordingly, no wrapper hook.
- `Showcase` composes `Slider` verbatim → inherits `"slider"`, plus its own
  `"showcase-group"` only for what it genuinely adds (per-slide grouping).
- Inside `RichText`, a tag that renders a real component (`a`, `img`,
  `video`, `btn`, `icon`) carries **no** `"richtext"` hook — it inherits
  that component's own (`"link"`, `"image"`, ...) instead. Only tags
  rendering plain HTML directly (`p`, headings, emphasis, lists, ...) carry
  `"richtext"` itself.

**This is a real, checkable rule, not a style preference**: before shipping
a new component that composes another, confirm it inherits the composed
component's hook rather than adding a parallel one — a duplicate hook on
the same element is the concrete failure mode this rule exists to prevent.

## The render-prop pattern, for arbitrary caller-owned content

Two components need to ARIA-wire an element they don't render themselves —
`cloneElement` can't do this safely, so both use a render-prop instead:

- **`Field.children`**: `(fieldProps) => ...` — wires an arbitrary native
  form control the caller owns.
- **`Popover`/`Tooltip`'s `trigger`**: same shape, but **no `ref` crosses
  the boundary** — the trigger element is found by querying a plain owned
  wrapper's `firstElementChild` fresh from the DOM (the same technique
  `Menu`'s own `toggleWrapperRef` already established), not by holding a
  ref reference across renders.

This is a deliberately small set — **only reach for a render-prop when a
component genuinely needs to ARIA-wire caller content it doesn't render
itself.** `Combobox`'s own input is a counter-example: even though it looks
similar to `Popover`'s trigger problem, the input isn't arbitrary caller
content (this component owns and renders it directly), so it takes no
render-prop at all — confirm which shape actually applies before defaulting
to a render-prop out of habit.

A THIRD, differently-motivated render-prop since exists — **`Menu.visual:
() => Node`** — worth distinguishing from the two above rather than folding
in as a third instance of the same reason. `Menu.visual` isn't ARIA-wiring
anything (it takes no arguments to merge onto the returned element, unlike
`Field.children`'s `fieldProps` or `Popover`/`Tooltip`'s event-handler
spread) — it exists to avoid giving `Menu`'s own module a static import of
`Image`/`ImgButton` (both real, `@zanix/space`-dependent components), since
a static ES import is unconditionally hoisted regardless of runtime
branching and would have made every `Menu` instance unusable inside a
`'use comet'` file even when a given instance never used an image-shaped
visual at all (see `space-ui-architecture`'s "Export surface" section and
`components/Menu/index.ts`'s own doc). The caller resolves the visual
element themselves (server-side, outside any Comet) and hands down the
already-built result. **The real, generalizable lesson**: a render-prop is
the right tool whenever a component needs a caller-supplied ELEMENT it
doesn't fully control the construction of — whether the reason is ARIA-wiring
(the original two) or avoiding a static dependency the component's own
module would otherwise carry unconditionally (`Menu.visual`) — not only the
narrower "ARIA-wire caller content" framing this section originally used
exclusively. Still don't reach for one without one of these two concrete
reasons.

## Real bugs already found and fixed — check before repeating the shape

- **A `useEffect` dependency on a fresh object literal caused an infinite
  render loop** (`usePosition`, building `Popover`) — fixed by keying on
  `JSON.stringify(options)` instead of the raw `options` object. Any new
  effect depending on an options/config object passed fresh on every render
  needs the same treatment, not just this one call site.
- **`clearInterval` was paired with a `setTimeout`** (found auditing the
  predecessor `Toast` implementation this component rescues real behavior
  from) — the mismatched pairing silently failed to cancel the scheduled
  auto-dismiss. Fixed with the matching `clearTimeout`, verified by a
  deterministic fake-clock test proving the pairing actually works — a
  timer cancellation claim needs a real test proving cancellation happens,
  not just that the code compiles.
- **React's `onChange` fires per-keystroke; Preact's own `onChange` means
  the literal native `change` event (fires only on blur)** (`Combobox`) — a
  real, confirmed divergence between the two renderers' event systems,
  caught by a failing test, fixed by using `onInput` specifically in the
  Preact binding. Don't assume an event name means the same thing across
  both renderer bindings — verify empirically per event, the same way this
  one was.
- **A document-level `Escape` listener reopened itself** (`Tooltip`) — an
  early attempt used the same `createEscapeToCloseHandler` shape `Popover`
  uses (attached to the trigger, with a refocus side effect), but the
  refocus call fires a genuine `focusin` event when the trigger wasn't
  already focused (the hover-only case) — `Tooltip`'s own `onFocus` handler
  then treated that synthetic focus as "open," reopening it immediately. A
  real test caught this. Fixed by listening on `document` instead, with no
  refocus side effect. **The lesson generalizes**: a "close and refocus the
  trigger" pattern is only safe when the trigger is guaranteed to have held
  real focus already — a hover-only-opened component needs the
  no-refocus variant instead.
- **An off-by-one in gap-collapsing logic** (`Pagination`'s windowing) —
  collapsing a 1-page gap into `'…'` saves no space over just showing that
  page directly; caught during testing, not assumed correct on the first
  try. A collapsing/windowing algorithm needs a test for the boundary case
  (a gap of exactly 1), not just the general case.
- **`Modal`/`Drawer` imported `@zanix/utils/logger` (the full,
  server-capable entry) directly for one `logger.warn` call, pulling remote
  `https://jsr.io/...` fetches into every browser bundle** — see
  `space-ui-architecture`'s "A second cross-package hazard" section for the
  full mechanism and the fix (`shared/client-logger.ts`). Any new component
  that needs to log anything follows the same rule, not just these two.
- **A real `<img>` (or any element with an image-fetching attribute — a
  `<video poster>`, a future `<picture>`-based component's own `<source>`)
  rendered with no `crossOrigin` escape hatch is a real, confirmed
  session-cookie hazard, not a theoretical one** — found via a live,
  reproduced bug in a real consumer: a profile-photo
  `<img>` pointed at a cross-origin asset host that happened to share the
  viewer's own hostname but a different port (a common same-machine-
  different-service dev/prod topology). Browsers attach cookies to an image
  request ambiently, with no CORS preflight, based on the target HOST alone
  — cookies are never port-scoped — so the calling app's own session cookies
  reached that other service, and ITS OWN response emitted a `Set-Cookie`
  under the ecosystem's shared cookie names that clobbered the viewing app's
  real session, silently logging the user out. `crossOrigin='anonymous'`
  (no cookies sent, a CORS response required instead) is the fix, and it has
  to be a prop the CALLER can set — the component itself has no way to know
  whether its own `src` is same- or cross-origin at the point it's given
  one. Fixed in `Image`/`Avatar` first, then audited across the rest of the
  package once a second gap search request surfaced two more real,
  unfixed instances: `Video` (native `<video crossorigin>` governs the
  `poster` fetch too, per the WHATWG spec — no opt-out existed) and
  `SocialNetworks`' image-logo variant (the one component in the whole
  package that builds a raw `h('img', …)` directly rather than composing
  `Image`). **The generalized rule for every future component**: any prop
  that becomes a real `src` on an `<img>`/`<video>`/`<audio>`/`<source>` (or
  any other element the HTML spec lets carry a `crossorigin` attribute)
  needs its own `crossOrigin?: 'anonymous' | 'use-credentials'` prop,
  forwarded unchanged onto that same native attribute, omitted by default so
  no caller is forced into a CORS requirement their host doesn't actually
  meet. `Card`/`ImgButton` needed no separate fix — both already spread a
  full `Omit<ImageProps, 'alt'>` object into their own internal `Image({...})`
  call, so `crossOrigin` reaches through automatically once `Image` itself
  supports it, with zero code change on their end; `RichText`'s `img`/`video`
  tags are the same story (an untyped `<props>` bag spread verbatim into
  `Image`/`Video`), just underdocumented rather than actually broken. Check
  every new `Image`/`Video`/`Avatar`/`Card`/`ImgButton`-composing component
  for this before shipping, not just ones that look image-shaped at a
  glance.

## Checklist before adding a new component

- [ ] **Default to the shareable-body `render.ts` pattern for a stateful
      component — a full second per-renderer implementation is the FALLBACK,
      never the default.** Write the body once, injecting the hooks it needs
      alongside `h` (see "Three implementation shapes" above), then check for
      a real, CONFIRMED divergence (not an assumed one) before splitting it —
      `Combobox`'s `onChange`/`onInput` is the only kind of thing that
      actually forces a split; "it uses `useEffect`" or "it's stateful" does
      not, on their own (`Counter`/`Table`/`Modal`/`Toast`/`Select`/`Tabs`/
      `Disclosure`/`RadioGroup`/`Drawer`/`Menu`/`Pagination` all turned out
      to be fully shareable once actually checked, not assumed to need a
      split because they have hooks). A full second implementation without
      a checked, named divergence is itself a finding worth flagging, not a
      safe default to fall back on out of habit.
- [ ] Controlled prop + callback exists for all real state, with an
      uncontrolled `defaultX` fallback for the simple case?
- [ ] If composing another real component, does it inherit that component's
      `data-space-ui` hook rather than adding a redundant one?
- [ ] If ARIA-wiring caller-owned content, does it genuinely need a
      render-prop — or does it actually own and render the content itself
      (no render-prop needed, like `Combobox`'s input)?
- [ ] Does any effect depend on an object/array passed fresh on every
      render? If so, is the dependency keyed on a stable derived value
      (`JSON.stringify`, a primitive), not the object itself?
- [ ] Does a timer-cancellation claim have a real test proving cancellation
      happens (a fake-clock test), not just that the code compiles?
- [ ] Has a cross-renderer event name (`onChange` and similar) been verified
      empirically to mean the same thing in both React and Preact bindings,
      not assumed?
- [ ] Does a "close and refocus" pattern assume the trigger held real focus
      — checked against whether this component can be hover-only-opened?
- [ ] Does this component (directly, or by composing another component that
      does) reach `@zanix/space` or any other real cross-package runtime
      dependency? See `space-ui-architecture`'s own "Export surface" section
      for the full rule and why it matters — if yes, its exports belong in
      their OWN new `./runtime/<kebab-name>`/`./runtime/<kebab-name>/preact`
      subpath, never the root barrel and never an existing component's own
      `./runtime/*` file or any shared combined barrel.
- [ ] Does this component render an `<img>`/`<video>`/`<audio>`/`<source>`
      (or compose one that does) from a caller-supplied `src`? If so, does it
      expose its own `crossOrigin?: 'anonymous' | 'use-credentials'`,
      forwarded unchanged onto that native attribute — see the real,
      confirmed session-cookie hazard documented above (`Image`/`Avatar`/
      `Video`/`SocialNetworks`) before assuming a new image-rendering
      component doesn't need one.
- [ ] Does this component log anything? See `space-ui-architecture`'s "A
      second cross-package hazard" section — `shared/client-logger.ts`,
      never `@zanix/utils/logger` directly.

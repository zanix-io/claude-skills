---
name: app-behaviors-and-overrides
description: ctx.behavior()/resolveBehavior() — pure function/strategy overrides a host can supply without forking the base app, the style-only-override pattern for swapping just a component's presentation, and the Configuration vs Extension vs Override decision table for picking the right mechanism. Use when a base app needs to expose something a host can customize without copying its code.
---

Covers `@zanix/app`'s `behaviors` mechanism — for customizing an app
without forking it or copying its code. For `resources` (the mechanism for
anything with a real lifecycle instead of a pure function), see
`app-manifest-and-composition`. File:line references point at
`~/Documents/Development/ZanixLibraries/app` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- Check the Configuration vs Extension vs Override table below before
  picking a mechanism — most "which one do I use" questions have exactly
  one right answer once the actual question (one value? a service with
  lifecycle? a pure function? genuinely new behavior?) is named precisely.
- A style-only override is the SAME registry as a full behavior override —
  don't look for a second API; the difference is what the component itself
  resolves for.

## `ctx.behavior()`: swapping a pure function/strategy

```ts
// The base app — never forked, never copied.
const billing = defineZanixApp({
  name: 'billing',
  behaviors: {
    calculateDiscount: {
      default: (order: Order) => order.total * 0, // no discount by default
      description: 'Calculates the discount applied to an order before checkout.',
    },
  },
  setup: async (ctx) => {
    const discount = ctx.behavior<(order: Order) => number>('calculateDiscount')(order)
  },
})

// The host composing it — overrides only the one thing that actually varies.
await activateApps([billing], {}, [], undefined, {}, [
  { appName: 'billing', name: 'calculateDiscount', implementation: (order) => order.total * 0.1 },
])
```

`ctx.behavior<T = unknown>(name)` resolves to the host-supplied override if
one was given, else the manifest's own `behaviors.<name>.default`, else
`undefined` — the same "override, else default" precedence
`ctx.config.get` already follows for its own Config Plane overlay. `T` is
manually specified, not inferred (`name` is just a string, with no real
argument like a class reference to infer it from) — exactly as sound as an
`as T` cast, never more; its only purpose is letting `ctx.behavior<T>(name)
?? default` type-check without wrapping the whole expression in an
external cast.

`activateApps`'s own `behaviors` parameter (`BehaviorOverride[]`) validates
eagerly — it throws `InternalError` *before anything else is constructed*
if an override names an app not in this activation, or a behavior name that
app never declared, the same fail-fast posture `validate()` already has for
`uses` naming an unknown `dependencies` slot.

Use `behaviors` for a PURE function/strategy a host might replace (a
pricing rule, a formatting strategy, a routing decision); reach for
`resources` instead when the swappable thing needs a real lifecycle
(construction, `close()`, health-gating, quotas).

## `resolveBehavior()`: overriding a single Comet/component from anywhere

`ctx.behavior()` only exists inside a `RuntimeContext`
(`setup`/`onStart`/`onStop`/`operations`) — a `@zanix/space` page's own
render has none of that. `resolveBehavior(appName, name)` resolves the SAME
registry standalone, from anywhere:

```tsx
// products/[id]/page.tsx — the BASE app's own page, never touched, never aware an override exists.
import { resolveBehavior } from '@zanix/app/runtime'
import DefaultAddToCartButton from './add-to-cart.tsx'

export default function ProductPage() {
  const AddToCartButton = resolveBehavior<typeof DefaultAddToCartButton>('shop', 'AddToCartButton')
    ?? DefaultAddToCartButton
  return <AddToCartButton product={product} />
}
```

A Comet is, structurally, just a function (props in, output out) —
registering one as a `behaviors` default needs no special framework
integration, and `@zanix/app` gains no Preact/React dependency either way:
`behaviors`/`resolveBehavior` never inspect what the function returns.
`ctx.behavior()` delegates to `resolveBehavior()` internally, so the two
entry points can never resolve differently.

## Style-only overrides: swap presentation without reimplementing logic

Replacing the WHOLE component (as above) is right when a host genuinely
wants different behavior — but it's the wrong tool when a host only wants a
different look and would otherwise have to copy the base component's own
logic just to attach a different className, exactly the duplication this
mechanism exists to avoid.

**This is not a second registry or a parallel API** — it's the same
`behaviors`/`resolveBehavior`, with the component itself (not the host, not
the page) resolving a style-shaped value for its own className/style prop,
instead of resolving an entire replacement component:

```tsx
// add-to-cart.tsx — the BASE app's own component, authored to expose its OWN style as swappable.
import { resolveBehavior } from '@zanix/app/runtime'

export default function AddToCartButton(props: AddToCartProps) {
  const className = resolveBehavior<string>('shop', 'AddToCartButtonClassName') ?? 'btn-default'
  // Every bit of logic below is still the base app's own — a host overriding the className above
  // never touches (and never has to reimplement) any of it.
  return <button className={className} disabled={props.disabled} onClick={props.onAdd}>Add to cart</button>
}
```

```ts
// The host — overrides ONLY the className, on every page rendering this component, without
// touching a single page and without reimplementing the button's click/disabled logic.
await activateApps([shopApp], {}, [], undefined, {}, [
  { appName: 'shop', name: 'AddToCartButtonClassName', implementation: () => 'btn-custom' },
])
```

Declare a `behaviors` entry whose default is a function returning the value
(a className string, a style object — whatever shape the component itself
expects), and have the COMPONENT resolve it for its own prop, rather than
having a page resolve an entire replacement component.

**One real precondition, not a gap**: this only works for a component whose
author already added the `resolveBehavior(...)` call for its own style
value — same precondition every `behaviors` slot already has (a host can
never override something the base app's author never declared as a slot in
the first place). It does NOT retroactively make an arbitrary,
already-written component's style overridable from outside; the base
component has to opt in.

## Configuration vs Extension vs Override

| Question | Mechanism |
| --- | --- |
| Change ONE VALUE, no code involved? | `config` |
| Replace a SERVICE with a real lifecycle (a connection, an authenticated client)? | `resources`/`dependencies`/`uses` + `registerResourceType` |
| Replace a PURE FUNCTION (no lifecycle)? | `behaviors` + `ctx.behavior()` |
| Add NEW behavior that doesn't replace anything existing? | A second Zanix App, composed alongside the base one — never a fork of the base app |

`resources` and `behaviors` look similar (both are host-suppliable
overrides resolved through `ctx`), but exist for genuinely different needs
— forcing a pure function through `resources` (a factory that just returns
a closure, never opened/closed/health-checked) works, but adds
construction/registry machinery a plain function never needed.

## Checklist before adding a new override point

- [ ] Does the swappable thing genuinely have no lifecycle — if it needs
      construction/`close()`/health-gating, it belongs in `resources`, not
      `behaviors`?
- [ ] Has the base component/function actually opted in by calling
      `resolveBehavior(...)`/`ctx.behavior(...)` itself — an override can
      never retroactively apply to code that never declared the slot?
- [ ] For a style-only need, does the component resolve its OWN style
      value, rather than the host/page resolving a whole replacement
      component?
- [ ] Would adding genuinely new behavior (not replacing anything) be
      better served by composing a second Zanix App instead of stretching
      `behaviors` to fit?

---
name: zanix-remote-api-app-pattern
description: The layered pattern for a @zanix/space consumer app that owns no data of its own and builds its own UI against a remote, typed Zanix API — a resource descriptor bound to the backend's real RTOs, a thin client over RestClient, file-based pages as the only HTTP-touching layer, and generic @zanix/space-ui composition. Grounded in @zanix/console's own real Triggers and Templates vertical slices. Use when scaffolding or reviewing a @zanix/space app that consumes another service's admin/API surface rather than owning its own backend.
---

This documents a real, proven shape — not a proposal. `@zanix/console` is a
`@zanix/space` app that owns no database of its own and presents a UI over
`@zanix/admin`'s remote hub API. Two independent vertical slices built this
way — Triggers (`src/triggers/`, `src/space/routes/triggers/`) and Templates
(`src/templates/`, `src/space/routes/templates/`) — each with its own full
CRUD flow (list, detail, create, edit, delete-with-confirmation) and its own
passing unit test suite (`src/@tests/unit/admin-resources/`,
`src/@tests/unit/clients/`, `src/@tests/unit/triggers/`,
`src/@tests/unit/templates/`). Templates is the proof the pattern
generalizes: it reused the exact hub-client-factory shape, the exact
resource-descriptor shape, and even the exact confirm-modal component
Triggers already established, unrenamed — zero new abstraction needed for
the second resource.

**`@zanix/console` is the cited precedent, not this skill's subject.** This
skill documents the reusable pattern any consumer app can follow when it
builds UI against a remote, typed Zanix backend — a fresh consumer names its
own layers after its own domain, not after `AdminResource`/`admin-resources/`
(see the naming note below).

## Golden rule (token savings)

- The six layers below are additive, in order — scaffold, then descriptor,
  then client, then pages, then presentation, then auth. Don't reach for a
  later layer's concern (e.g. a page-level auth guard) while still deciding
  the descriptor's shape.
- Layers 1-2 (the backend's real API contract, and whatever generated/typed
  client artifact the CLI produces from it) are shared, published
  infrastructure — never redesign them per consumer. Layers 3-5 (the thin
  client wrapper and its fronting interactor, the pages, the presentation
  composition) are where a consumer's own domain work actually lives —
  expect to write these fresh for every new resource, the same way
  Templates did after Triggers.
- Read `triggers.resource.ts` alongside `templates.resource.ts`
  (`console/src/triggers/`, `console/src/templates/`) side by side before
  designing a new resource descriptor — the two real, documented deviations
  between them (below) are exactly the kind of judgment call a new resource
  will also have to make.

## The six layers

### 1. Scaffold base

`zanix new space` (add `space-server` instead only if the consumer also
serves its own API alongside consuming a remote one). Nothing about this
pattern changes that flow — the app starts as an ordinary `@zanix/space`
project.

### 2. Resource descriptor — one file per resource, naming is yours

A resource descriptor is a plain object binding **presentation/interaction
metadata** (which columns, which actions, which confirmation copy) to the
backend's **real RTO(s)** — never redeclaring a field's type, requirement, or
validation constraints locally. `@zanix/console`'s own shape
(`src/admin-resources/admin-resource.ts`) is the concrete reference:

```ts
export interface AdminResourceField {
  key: string // must be a real property of the referenced RTO(s)
  label: string
  column?: { order: number; format?: 'text' | 'badge' | 'date' | 'bool' }
  detail?: { order: number; editable: boolean }
}

export interface AdminResourceAction {
  name: string // 'create' | 'update' | 'remove' | 'sync' | a custom name
  method: HttpMethod
  permission: string // reused unchanged from the backend's own auth package
  confirm?: { title: string; body: string }
}
```

**`actions[].permission` is documentation-and-drift-detection metadata, not
itself the enforcement mechanism — don't assume declaring it here wires
anything up.** In both real resources, real enforcement happens on the page
(layer 4), via a `@Guard(requireAdminSession([ADMIN_ROLE,
ADMIN_TRIGGERS_ROLE]))`-shaped guard that hardcodes the same permission
constant a second time — `permission` here is never read at runtime by
anything (confirmed: no reference to `.permission` anywhere in
`@zanix/console`'s own source). Unlike `confirm`, which the delete page
genuinely reads (`REMOVE_ACTION.confirm.title`/`.body`) to render its
comet's copy, `permission` is currently a second, independent copy of the
same constant that can silently drift from the guard actually enforcing it.
Keep both copies in sync by hand until/unless a future revision has the
page's own guard read `permission` from the descriptor directly instead of
repeating the constant — don't treat declaring it here as equivalent to
gating anything.

```ts
export interface AdminResource {
  name: string
  rto: { list?: RtoTypes; body?: RtoTypes; params?: RtoTypes }
  fields: AdminResourceField[]
  actions: AdminResourceAction[]
}
```

**Naming is local to the consumer, not part of the pattern.** `@zanix/console`
calls this `AdminResource`/`admin-resources/` because its own domain is
admin-shaped — every resource it presents really is an admin resource. A
consumer building a UI over a non-admin API (a catalog, a billing dashboard,
anything not fronting `@zanix/admin`) names its own equivalent after its own
domain — `CatalogResource`/`catalog-resources/`, or whatever fits — not
`AdminResource`. Nothing about the shape above is admin-specific.

**Deliberately stays local to the consumer app, never generalized into a
shared package.** This was an explicit, already-made decision for
`@zanix/console` itself: no second real consumer existed yet even inside that
one app, and freezing a presentation-shaped type in a widely-consumed base
package is expensive to walk back speculatively. Revisit centralizing it only
once a second real, independent consumer needs the identical shape — not
before.

**Reference the backend's real RTO by field name, never redeclare it:**

- If the backend is a Zanix package and re-exports the RTO from a public
  subpath, import it directly. Triggers does this for its body RTO:
  `CreateTriggerRTO`, re-exported unchanged from `@zanix/datamaster/triggers-api`
  (`src/triggers/triggers.resource.ts`).
- If the backend's own equivalent type is exported **type-only**, not as a
  real runtime value, mirror its exact shape locally as a real class instead
  of reaching into the backend's internals for it. Triggers does exactly
  this for `TriggerServiceModelParamsRTO`
  (`src/triggers/rtos/trigger-service-model-params.rto.ts`): `@zanix/admin`'s
  `mod.ts` does export the name, but only via `export type { ... } from
  'modules/triggers/rtos/triggers.rto.ts'` — a type-only re-export erased at
  build time, with no real constructor a consumer can construct against.
  `classMetadata()` (the field-drift safeguard below) and any real
  `@Params()` decorator both need the actual runtime class, so a type-only
  export doesn't cover this layer's need even though the name technically
  appears in the package's public surface. Check a candidate import this
  way — `export type` vs. a real value `export` — rather than only grepping
  `mod.ts` for the name and assuming a hit means it's usable here. Templates
  needed no such local mirror — `@zanix/notifications/templates-api`
  re-exports its own `TemplateParamsRTO` as a real value
  (`export { CreateTemplateRTO, TemplateParamsRTO, UpdateTemplateRTO } from
  './rtos/templates.rto.ts'`) — which is the normal case; hand-mirroring is
  the fallback for a type-only export, not the default.
- A resource's `rto.body` doesn't have to point at the create RTO by
  convention — pick whichever real RTO's fields actually cover everything
  the resource declares in `fields`. Templates' own descriptor
  (`src/templates/templates.resource.ts`) points `rto.body` at
  `UpdateTemplateRTO`, not `CreateTemplateRTO`, specifically because create
  and update split fields differently there (`active` only exists on
  update) and the descriptor's `fields` needs `active` covered for the
  drift check below to pass. This is a genuine per-resource judgment call,
  not something to copy blindly from an earlier resource.
- Read-only/bookkeeping fields the backend computes itself and never accepts
  on any request RTO (`isDefault`/`lastSyncedTriggers` on Triggers;
  `source`/`version`/`hash`/`lastSyncedHbs` on Templates) stay out of
  `fields` entirely — adding one means inventing a new upstream RTO field,
  which is the backend's decision, not the consumer's workaround.

**Keep a real, automated safeguard against field drift.** `@zanix/console`'s
`assertAdminResourceFieldsMatchRto` (same file) walks the referenced RTO(s)
via `@zanix/validator`'s `classMetadata` and throws if any declared
`fields[].key` doesn't exist on the union of the resource's own referenced
RTOs. Call it from a unit test for every concrete resource a consumer
defines — see `src/@tests/unit/admin-resources/triggers-resource.test.ts`
and its Templates sibling for the reference usage. Without this, a
resource's presentation metadata can silently drift out of sync with the
real contract it claims to describe.

### 3. Thin client over `RestClient`

One wrapper module per backend surface — never a browser `fetch`, and never
called from anywhere but a page's `loader`/`action`. `@zanix/console`'s
`src/clients/triggers-hub.client.ts`, `templates-hub.client.ts`, and
`registry-hub.client.ts` are three real instances of the identical shape:

- A `*ClientFactory` type (`() => Client | Promise<Client>`).
- A module-level `activeFactory`, defaulting to a real factory that resolves
  auth headers and the backend's base URL, then constructs the real typed
  client (`TriggersHubClient`/`TemplatesHubClient`/`RegistryHubClient`, all
  imported from the backend's own published package — never hand-rolled).
- `set*ClientFactory`/`reset*ClientFactory` — the seam a test swaps in a fake
  client through, so the interactor's own test suite never exercises real
  crypto or network.
- A `get*Client()` accessor that returns whatever the active factory
  produces.

This shape is why adding Templates' client took no new design: it's
`triggers-hub.client.ts` copied and re-typed, nothing invented. Auth headers
for the client itself come from `src/clients/admin-hub-auth.ts` — see layer 6
below.

**A page never calls this client directly — a `ZanixInteractor` sits between
them.** `@zanix/console`'s `TriggersInteractor`/`TemplatesInteractor`
(`src/triggers/triggers.interactor.ts`, `src/templates/templates.interactor.ts`,
both `@Interactor()`-decorated `ZanixInteractor` subclasses from
`@zanix/server`) are what a page's `loader`/`action` actually calls
(`this.interactor.list()`/`.get()`/`.create()`/`.update()`/`.remove()`); the
interactor is what calls `get*Client()`. This is the same Handler/Provider
separation `zanix-server-conventions` documents generally
(`SpacePageController` is a Handler; Handlers never resolve
Providers/Connectors directly) — see that skill's own `## Interactors /
Services` section rather than re-deriving the rule here. The interactor is
also where destructive-action audit logging belongs: both real interactors
log a persisted `logger.info` entry before `update`/`remove` actually run,
never inside the page itself.

### 4. Pages — the only layer touching HTTP, guards, and the interactor

`@zanix/space` file-based routes: `loader` for reads, `action` for
mutations. This is the sole layer permitted to call a layer-3 interactor or
apply an auth guard — resource descriptors and presentation components never
do either. `@zanix/console`'s two real slices both follow the same four-route
shape:

```
routes/<resource>/page.tsx                    — list (loader only)
routes/<resource>/new/page.tsx                — create (loader + action)
routes/<resource>/[id.../page.tsx             — detail + delete (loader + action)
routes/<resource>/[id.../edit/page.tsx        — update (loader + action)
```

(`src/space/routes/triggers/` and `src/space/routes/templates/` are the two
real instances — `[serviceId]/[model]` and `[channel]/[name]` respectively.)

A list page's columns and a detail page's fields are always *derived* from
the resource descriptor's own `fields` metadata (filtered by `column`/
`detail`, sorted by `order`) — never a second, hand-maintained parallel list.
Both real pages (`triggers/page.tsx`'s `COLUMNS`, `templates/page.tsx`'s
`COLUMNS`) do exactly this, so a descriptor change is the only edit a column
change ever needs.

**Real footgun: a path param's case must actually round-trip, and this needs
checking, not assuming.** `@zanix/server`'s router preserves each path
segment's real casing end to end — a mixed-case value in a param position
(an entity name like `Invoice`, a template name like
`Password-Recovery_V2`) comes back exactly as sent. Both real detail routes
rely on this directly (`[serviceId]/[model]/page.tsx`'s `model`,
`[channel]/[name]/page.tsx`'s `name`) — a genuine path-segment param, never a
query-string workaround for a value whose casing matters. When building a
new page that keys a resource by a case-sensitive identifier, use a real
route param for it and confirm case-preservation holds for the current
router version rather than assuming it — don't route around a suspected
case issue with a query string; that's a workaround for a problem the router
already solves correctly.

**Every page with a real mutating `action` also carries `@Guard(csrfGuard())`,
alongside the session guard, not instead of it.** All four real mutating
routes (`new/page.tsx`, the detail page's `remove` action, and `edit/page.tsx`,
for both Triggers and Templates) stack both guards on the class
(`@Guard(csrfGuard())` then `@Guard(requireAdminSession([...]))`); a
read-only list page carries neither. See `space-middleware-and-security` for
`csrfGuard()` itself — this pattern only says where it applies, not how it
works.

### 5. Presentation — generic `space-ui` plus consumer-specific composition

`@zanix/space-ui`'s `Table` (and other generic components) render the shape
layer 4 hands them; consumer-specific composition — how a resource's own
detail view lays out its fields, its own filter form — lives in the
consumer's own `components/`/`comets/`, never inside `space-ui` itself.
`space-ui` stays domain-agnostic; see `space-ui-component-patterns` for the
discipline that package's own components follow (not duplicated here).

**A confirm-action component built generically reuses across resources with
zero changes.** `@zanix/console`'s `src/space/comets/delete-trigger-modal.comet.tsx`
takes `title`/`body` as plain props — always sourced from the resource
descriptor's own `actions[].confirm` entry, never hardcoded per resource.
Templates' detail page (`src/space/routes/templates/[channel]/[name]/page.tsx`)
imports and reuses this exact file, unrenamed, for its own delete
confirmation — no `delete-template-modal.comet.tsx` was written. Design a
destructive-action confirmation comet this way from the start: generic over
its copy, composed from existing `space-ui` primitives (`Modal`/`Button`),
never hardcoding the first resource's domain language into a component a
second resource will need too.

### 6. Auth

Two distinct credential paths, never conflated:

- **Human session** — `@zanix/auth`'s `HttpOnly` refresh-token cookie, for
  the human operator navigating the app. A full-page `GET`/`<a>` navigation
  can't attach a bearer `Authorization` header the way `AuthTokenValidation`/
  `jwtValidationGuard` expect, so a `@zanix/space` page guard re-derives the
  session from the cookie per page load instead — see `@zanix/console`'s
  `src/auth/guards.ts` (`requireAdminSession`) for the real composition of
  `refreshSessionTokens` + `permissionsPipe`, applied as a class-level
  `@Guard(...)` on the page controller.
- **Server-to-server** — when a page's `loader`/`action` calls another Zanix
  service (not the human's own browser), use `createServiceAssertion`/
  `exchangeServiceCredential` (`@zanix/auth`'s `createServiceAuthClient`) to
  sign and exchange a service credential — never a token exposed to the
  browser. `@zanix/console`'s `src/auth/service-client.ts`
  (`consoleServiceAuthClient`) and `src/clients/admin-hub-auth.ts`
  (`getAdminHubAuthHeaders`) are the real wrapper and the real call site: a
  cached `(targetServiceId, exchangeUrl) => Promise<ServiceAuthHeaders>`
  function, rebuilt only when a test needs a fresh one
  (`resetAdminHubAuthCache`), never per request.

Both mechanisms are pre-existing `@zanix/auth` primitives — this pattern
composes them, it doesn't add a new auth mechanism of its own.

## What stays the same vs. what's genuinely new per consumer

Layers 1-2 — the backend's real API contract and whatever typed
client/RTO artifact is generated or published from it — are the exact same
shared infrastructure regardless of which consumer app is built. **Layers
3-5 are domain-specific by nature and are never auto-generated**: the thin
client wrapper and its fronting interactor, the pages, and the presentation
composition all encode choices specific to one consumer's own resources
(which fields matter on a list, what a destructive action's confirmation
copy says, how a filter form is shaped) that no generator can decide
correctly on a consumer's behalf — the same reasoning that keeps
`AdminResource` itself uncentralized (above) also rules out a full admin-UI
generator for these layers.

## Skills this pattern builds on

Load these directly rather than expecting this skill to restate their
content:

- `space-routing-and-rendering` — the file-based routing/layout mechanics
  layer 4's pages are built on.
- `space-ui-component-patterns` — the discipline for building or extending
  any `space-ui` component layer 5 composes.
- `zanix-server-conventions` — the general Handler/Interactor/Provider
  separation layer 3's interactor and layer 4's pages compose (`##
  Interactors / Services`), not restated here.
- `space-middleware-and-security` — `csrfGuard()`/`requireAdminSession`-style
  guard composition layer 4's mutating pages apply, and the CSP/security
  header defaults every page gets regardless.
- `admin-service-authentication` — the sign/exchange/call service-credential
  flow layer 6's server-to-server bullet describes, when the remote backend
  is itself `@zanix/admin`-shaped.

## Checklist before calling a new resource's vertical slice done

- [ ] Resource descriptor references the backend's real RTO(s) by field name
      only — no redeclared type/constraint/default anywhere in it. Confirmed
      it's a real, usable value export (not a type-only re-export) before
      importing it directly.
- [ ] A unit test calls the descriptor's own field-drift assertion
      (`assertAdminResourceFieldsMatchRto` or the consumer's equivalent)
      against every concrete resource.
- [ ] The thin client lives in its own module, follows the
      factory/set/reset/get shape, and is never called outside the
      interactor that fronts it.
- [ ] A `ZanixInteractor` sits between every page and the thin client — no
      page's `loader`/`action` calls a `get*Client()` accessor directly.
      Destructive-action audit logging lives in the interactor, not the page.
- [ ] List/detail views derive their columns/fields from the descriptor's
      own metadata — no second hand-maintained list.
- [ ] Any case-sensitive identifier used in a route is a real path
      segment, with case-preservation actually confirmed, not a
      query-string workaround.
- [ ] Every page with a real mutating `action` carries `@Guard(csrfGuard())`
      alongside its session guard — never one without the other.
- [ ] A destructive action's confirmation component takes its copy as
      props sourced from the descriptor — never hardcodes one resource's
      domain language — so a second resource can reuse it unchanged.
- [ ] Human-session and server-to-server auth are not conflated: no token
      meant for a service-to-service call is ever exposed to the browser.
- [ ] If `actions[].permission` is declared on the descriptor, its value is
      kept in sync by hand with whatever permission constant the page's own
      guard actually enforces — the descriptor doesn't wire the guard itself.

## Out of scope — do not do these

- Building a new agent (e.g. a `console-builder`-style generator) to
  automate this pattern end to end. That's explicitly sequenced AFTER this
  skill exists and after real infrastructure it would depend on
  (`zanix generate openapi`, a `space-ui` `Table` component, `@zanix/admin`'s
  registry endpoint) is in place — not before, and not part of this skill's
  own scope.
- Centralizing the resource-descriptor shape (`AdminResource` or a
  consumer's own equivalent) into `@zanix/utils` or any other shared
  package. That stays a per-consumer decision until a second real,
  independent consumer needs the identical shape.
- Designing a `zanix new --template <name>` scaffold preset for this
  pattern. Real, already-built preset infrastructure exists in `@zanix/cli`
  (`ScaffoldRecipeRegistry`), but adding a second preset is a product
  decision to make only after this pattern has run manually against more
  than one real case — not an architecture question this skill settles.
- The backend-side API contract itself (RTOs, controllers, Discovery,
  `ServiceRegistry`) — that's `zanix-local-api-vs-aggregator`/
  `zanix-local-api-implementation`'s job. This skill only covers how a
  consumer builds UI against an already-existing remote contract.

---
name: zanix-remote-api-app-pattern
description: The layered pattern for a @zanix/space consumer app that owns no data of its own and builds its own UI against a remote, typed Zanix API — a resource descriptor bound to the backend's real RTOs, a thin client over RestClient, file-based pages as the only HTTP-touching layer, and generic @zanix/space-ui composition. Grounded in @zanix/console's own real Triggers/Templates/DLQ vertical slices, and in @zanix/console-kit, the shared infrastructure extracted from them. Use when scaffolding or reviewing a @zanix/space app that consumes another service's admin/API surface rather than owning its own backend.
---

This documents a real, proven shape — not a proposal. `@zanix/console` is a
`@zanix/space` app that owns no database of its own and presents a UI over
`@zanix/admin`'s remote hub API. Three independent vertical slices built this
way — Triggers (`src/triggers/`, `src/space/routes/triggers/`), Templates
(`src/templates/`, `src/space/routes/templates/`), and DLQ (`src/dlq/`,
`src/space/routes/dlq/`, a narrower shape with no create/edit route — see
layer 4's own route-shape note) — each with its own passing test suite
(`src/@tests/unit/admin-resources/`, `src/@tests/unit/clients/`,
`src/@tests/unit/{triggers,templates,dlq}/`). Templates and DLQ are the proof
the pattern generalizes: both reused the exact hub-client-factory shape, the
exact resource-descriptor shape, and even the exact confirm-modal component
Triggers already established, unrenamed — zero new abstraction needed for
either later resource.

**`@zanix/console` is the cited precedent, not this skill's subject.** This
skill documents the reusable pattern any consumer app can follow when it
builds UI against a remote, typed Zanix backend — a fresh consumer names its
own layers after its own domain, not after `AdminResource`/`admin-resources/`
(see the naming note below).

**`@zanix/console-kit` now exists as real, shared infrastructure for parts of
this pattern** — extracted from `@zanix/console`'s own original
`admin-resources/`/`clients/`/`auth/` once a second real, independent
deployer of this pattern (its own `zanix-admin` hub, its own consumer app,
not yet built as of this writing) needed the identical shape instead of
forking `console`'s source. Not yet published to JSR — a consumer currently
links it locally (see `deno-workspace-link-pitfalls`). `console` remains the
first real consumer of the kit, and this pattern's own end-to-end validation
ground — not the kit itself. Each layer section below says explicitly what
moved into the kit and what's still per-consumer.

## Golden rule (token savings)

- The six layers below are additive, in order — scaffold, then descriptor,
  then client, then pages, then presentation, then auth. Don't reach for a
  later layer's concern (e.g. a page-level auth guard) while still deciding
  the descriptor's shape.
- Layer 1 (the backend's real API contract) is shared, published
  infrastructure — never redesign it per consumer. As of `@zanix/console-kit`'s
  extraction, the *mechanical shape* of layers 2 (the `AdminResource` type
  and its field-drift safeguard), 3 (the hub-client-factory triad), 6
  (auth composition), and a narrow slice of 4 (column/detail-field
  derivation) are ALSO shared, imported infrastructure now — a consumer
  installs/instantiates these, never redefines the shape from scratch. What
  stays genuinely per-consumer, in every layer, is the DOMAIN CONTENT poured
  into that shape: which concrete resource a descriptor describes, which
  concrete client class a factory wraps, which concrete columns/filters a
  page renders, which concrete service identity an auth instantiation binds
  to. Expect to write that content fresh for every new resource, the same
  way Templates did after Triggers — see "What stays the same vs. what's
  genuinely new per consumer" below for the precise line.
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

### 2. Resource descriptor — one instance per resource, naming the CONTENT is yours

A resource descriptor is a plain object binding **presentation/interaction
metadata** (which columns, which actions, which confirmation copy) to the
backend's **real RTO(s)** — never redeclaring a field's type, requirement, or
validation constraints locally. The `AdminResource` TYPE itself is now
`@zanix/console-kit`'s own shared shape (its root export,
`src/admin-resource.ts` in that package) — a consumer imports it rather than
redefining it locally; the concrete shape:

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

**The CONCRETE INSTANCE's name is local to the consumer; the TYPE's name is
not, now that it's centralized.** `@zanix/console` calls its own instances
`TRIGGERS_ADMIN_RESOURCE`/`TEMPLATES_ADMIN_RESOURCE` — that half stays a
free per-resource choice, the same as any other constant. The `AdminResource`
type itself, though, is now a real, shared export from `@zanix/console-kit`
(named `AdminResource` because `console`'s own domain — and the kit's own
name — are admin-shaped), not something a consumer redeclares locally
anymore. A consumer building UI over a genuinely non-admin API (a catalog, a
billing dashboard, anything not fronting `@zanix/admin`) that wants a
domain-fitting local name can still alias it on import
(`import { AdminResource as CatalogResource } from '@zanix/console-kit'`) —
nothing about the shape itself is admin-specific — but the type it aliases
is shared, not a second, independently-defined interface. This is a real,
deliberate trade-off the centralization below introduced, not an oversight.

**Centralized into `@zanix/console-kit` once a second real, independent
consumer needed the identical shape.** This was previously an explicit,
deliberately-deferred decision — freezing a presentation-shaped type in a
shared package before a genuine second consumer existed would have been
speculative. That precondition is now met (see this skill's own intro), so
the type lives in the kit; `assertAdminResourceFieldsMatchRto` and
`formatFieldValue` moved with it (same file in the kit,
`columnsFromResource`/`detailFieldsFromResource` alongside them — see layer 4
below). A NEW consumer never redefines `AdminResource` locally — it imports
the kit's own type and only ever writes concrete instances against it.

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

**Keep a real, automated safeguard against field drift.** `@zanix/console-kit`'s
own `assertAdminResourceFieldsMatchRto` (root export, same module as
`AdminResource` itself) walks the referenced RTO(s) via `@zanix/validator`'s
`classMetadata` and throws if any declared `fields[].key` doesn't exist on
the union of the resource's own referenced RTOs. Call it from a unit test
for every concrete resource a consumer defines — see
`src/@tests/unit/admin-resources/triggers-resource.test.ts` in `@zanix/console`
for the reference usage against a real resource, and the kit's own
`src/@tests/unit/admin-resource.test.ts` for the mechanism's own generic
drift-detection tests (moved there from `console`'s test suite once the
mechanism itself did). Without this, a resource's presentation metadata can
silently drift out of sync with the real contract it claims to describe.

### 3. Thin client over `RestClient`

One wrapper module per backend surface — never a browser `fetch`, and never
called from anywhere but a page's `loader`/`action`. The pluggable-factory
SHAPE is now `@zanix/console-kit/client`'s own `createHubClientFactory`
(`(ClientCtor, hubAuth) => {get, set, reset}`) — a consumer instantiates it
once per hub-facing resource client, rather than hand-writing the shape
fresh:

```ts
const { get, set, reset } = createHubClientFactory(TriggersHubClient, {
  requireAdminHubBaseUrl,
  getAdminHubAuthHeaders,
})
export { get as getTriggersHubClient, set as setTriggersHubClientFactory, reset as resetTriggersHubClientFactory }
```

- `get` returns the real client, authenticated against the configured hub,
  via whichever factory `set` last installed (or the real default).
- `set`/`reset` are the seam a test swaps in a fake client through, so the
  interactor's own test suite never exercises real crypto or network.

`@zanix/console`'s `src/clients/triggers-hub.client.ts`/
`templates-hub.client.ts`/`registry-hub.client.ts`/`dlq-hub.client.ts` are
four real instantiations of this ONE shared shape — before this extraction,
they were four hand-written copies differing only in which client class
they constructed; confirmed byte-for-byte identical otherwise. Adding a
new resource's client is now genuinely zero new design: call
`createHubClientFactory` with the new client class, nothing to invent. The
`hubAuth` pair (`requireAdminHubBaseUrl`/`getAdminHubAuthHeaders`) comes from
layer 6's own `createAdminHubAuthClient` instantiation, below.

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
do either. Triggers and Templates both follow the same four-route shape:

```
routes/<resource>/page.tsx                    — list (loader only)
routes/<resource>/new/page.tsx                — create (loader + action)
routes/<resource>/[id.../page.tsx             — detail + delete (loader + action)
routes/<resource>/[id.../edit/page.tsx        — update (loader + action)
```

(`src/space/routes/triggers/` and `src/space/routes/templates/` are the two
real instances — `[serviceId]/[model]` and `[channel]/[name]` respectively.)

**This four-route shape isn't universal — a resource with a genuinely
different action set gets a genuinely different route count, not a forced
fit.** DLQ (`src/space/routes/dlq/`) has only two routes — `page.tsx` (list)
and `[serviceId]/[id]/page.tsx` (detail) — no `new/`, no `edit/`, because
DLQ's own two real mutations (`requeue`/`discard`) are both destructive
actions on an EXISTING entry, never a create/update in the CRUD sense. Both
share the ONE detail route's own `action`, distinguished by a hidden
`<input name="_intent">` value on each of the two forms rather than by a
separate route per mutation — a legitimate variant of layer 4, not a
deviation to normalize away. Decide a new resource's own route count from
its real action set, not from copying the four-route shape by default.

A list page's columns and a detail page's fields are always *derived* from
the resource descriptor's own `fields` metadata (filtered by `column`/
`detail`, sorted by `order`) — never a second, hand-maintained parallel list.
The derivation ITSELF is now `@zanix/console-kit`'s own
`columnsFromResource<TRow>(resource)`/`detailFieldsFromResource(resource)`
(root exports, alongside `AdminResource`) — extracted once this exact
`.filter().sort().map()` shape turned up byte-for-byte identical across
every real list/detail page (`triggers/page.tsx`, `templates/page.tsx`,
`dlq/page.tsx`'s own `COLUMNS`; the two real detail pages' own
`DETAIL_FIELDS`), differing only in the row's own type parameter. A
descriptor change is the only edit a column change ever needs; a new
resource's list/detail page calls these two functions rather than
re-deriving the shape.

**Deliberately NOT extended into a full generic page-shell factory** (e.g. a
`createResourceListPage(resource, ...)` that would wire an entire
`SpacePageController`, guard included). Investigated as technically
feasible — `@zanix/space`'s `@Page()`/`@Guard()` are pure runtime class
decorators, with no static analysis standing in the way of a factory
function declaring one internally — but a real page's OWN content beyond its
columns (filter forms, table captions, empty states, row-href patterns) is
genuinely domain-specific per resource, confirmed by reading every real
page rather than assumed; templating it would either lose that flexibility
or amount to rebuilding a full admin generator, which stays out of scope
(below). Everything past `columnsFromResource`/`detailFieldsFromResource`
stays hand-written per page, by design.

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
alongside the session guard, not instead of it.** Every real mutating route
across all three slices (`new/page.tsx`, the detail page's own destructive
action(s), and `edit/page.tsx` where one exists) stacks both guards on the
class (`@Guard(csrfGuard())` then `@Guard(requireAdminSession([...]))`) —
confirmed true even for DLQ's own two-intent single detail route (above),
which has no `edit/page.tsx` at all; a read-only list page carries neither.
See `space-middleware-and-security` for `csrfGuard()` itself — this pattern
only says where it applies, not how it
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
  session from the cookie per page load instead — see `@zanix/console-kit/auth`'s
  `requireAdminSession` (the SHARED implementation now — `@zanix/console`'s
  own `src/auth/guards.ts` is a thin re-export, no local logic left) for the
  real composition of `refreshSessionTokens` + `permissionsPipe`, applied as
  a class-level
  `@Guard(...)` on the page controller. That guard shape — rotation followed
  by a later check in the same chain — is exactly the case
  `attachRotatedSessionToError`/`recoverRotatedSessionCookie` exist for
  (`auth-jwt-and-sessions`'s "Guard-stage rotation recovery"): without it, a
  403 from `permissionsPipe` strands the client on a cookie the rotation
  already blocklisted, with the replacement never delivered.
  `requireAdminSession` wraps its `requirePermissions` throw in
  `attachRotatedSessionToError`, and this project's own `mod.ts` wires
  `onError: globalErrorHandler(recoverRotatedSessionCookie(),
  createNotFoundHandler())` (`space-routing-and-rendering`'s "Composing
  multiple `onError` handlers") to recover it. Any consumer building a
  human-session guard on this same `refreshSessionTokens` + later-check
  shape needs both halves wired, not just the guard side.
- **Server-to-server** — when a page's `loader`/`action` calls another Zanix
  service (not the human's own browser), use `createServiceAssertion`/
  `exchangeServiceCredential` (`@zanix/auth`'s `createServiceAuthClient`) to
  sign and exchange a service credential — never a token exposed to the
  browser. `@zanix/console-kit/auth`'s `createHubServiceAuthClient(serviceId)`
  (a thin wrapper) and `createAdminHubAuthClient(serviceId)` (the real
  credential wiring: `requireAdminHubBaseUrl`/`getAdminHubAuthHeaders`/
  `resetAdminHubAuthCache`) are the shared implementation now — a consumer
  instantiates `createAdminHubAuthClient` ONCE, bound to its own service
  identity (`@zanix/console`'s own `src/clients/admin-hub-auth.ts` does
  exactly this, bound to `CONSOLE_SERVICE_ID` — see layer 3 above for how the
  hub-client factories consume its two accessors). `getAdminHubAuthHeaders`
  itself is a cached `() => Promise<ServiceAuthHeaders>` function, rebuilt
  only when a test needs a fresh one (`resetAdminHubAuthCache`), never per
  request.

Both mechanisms are pre-existing `@zanix/auth` primitives — this pattern
composes them, it doesn't add a new auth mechanism of its own.

## What stays the same vs. what's genuinely new per consumer

**The distinction is no longer "which layer" — it's "shape vs. content"
within every layer.** Layer 1 (the backend's real API contract) is the exact
same shared infrastructure regardless of which consumer app is built, same
as always. As of `@zanix/console-kit`'s extraction, the reusable MECHANICAL
SHAPE of layers 2, 3, 6, and a slice of 4 is ALSO shared — a consumer
imports/instantiates it, never reinvents it (see each layer's own section
above for exactly what moved). What's still genuinely new per consumer, in
every one of those layers, is the DOMAIN CONTENT poured into that shared
shape: which concrete resource a descriptor describes, which concrete client
class a factory wraps, which concrete service identity an auth
instantiation binds to, which concrete columns/filters/copy a page renders.
**No generator can decide that content correctly on a consumer's behalf** —
confirmed directly: a full generic page-shell factory (layer 4) was
investigated as technically feasible and deliberately not built, precisely
because a real page's own filter forms/captions/empty states turned out to
be genuinely domain-specific, not boilerplate, when checked against every
real page rather than assumed. That's the same reasoning that ruled out a
full admin-UI generator for the pattern as a whole (below) — this
extraction moved the genuinely-duplicated SHAPE into the kit; it didn't
change which parts of the pattern are inherently per-consumer.

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
- `auth-jwt-and-sessions` — the refresh-rotation mechanics and "Guard-stage
  rotation recovery" (`attachRotatedSessionToError`/
  `recoverRotatedSessionCookie`) the human-session bullet above composes,
  for any guard combining rotation with a later check.
- `admin-service-authentication` — the sign/exchange/call service-credential
  flow layer 6's server-to-server bullet describes, when the remote backend
  is itself `@zanix/admin`-shaped.
- `deno-workspace-link-pitfalls` — how to link an unpublished
  `@zanix/console-kit` checkout locally before it's on JSR (`@zanix/console`'s
  own `deno.jsonc` does this today) without hitting one of the four real,
  confirmed footguns that technique carries.

To prove a real instance of this pattern still works end to end — not just
that its unit tests pass against a mocked hub client — see
`zanix-remote-api-app-e2e-validation`'s own runbook: three real processes
(business service, admin hub, consumer app), driven with `curl` through a
full CRUD cycle. That's a maintainer-only validation task, not part of
building a new resource.

## Checklist before calling a new resource's vertical slice done

- [ ] Resource descriptor uses `@zanix/console-kit`'s own `AdminResource`
      type (never a locally-redeclared interface) and references the
      backend's real RTO(s) by field name only — no redeclared
      type/constraint/default anywhere in it. Confirmed it's a real, usable
      value export (not a type-only re-export) before importing it directly.
- [ ] A unit test calls `@zanix/console-kit`'s own
      `assertAdminResourceFieldsMatchRto` against every concrete resource.
- [ ] The thin client is one instantiation of `@zanix/console-kit/client`'s
      `createHubClientFactory`, and is never called outside the interactor
      that fronts it.
- [ ] A `ZanixInteractor` sits between every page and the thin client — no
      page's `loader`/`action` calls a `get*Client()` accessor directly.
      Destructive-action audit logging lives in the interactor, not the page.
- [ ] List/detail views derive their columns/fields via
      `@zanix/console-kit`'s own `columnsFromResource`/
      `detailFieldsFromResource` — no second hand-maintained list.
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
  automate this pattern end to end. Two of the three prerequisites this was
  sequenced after are now real and confirmed (`space-ui`'s `Table`
  component; `@zanix/admin`'s `GET /registry` endpoint) — but the third,
  `zanix generate openapi`, only generates in the REVERSE direction
  (introspects a project's own REST routes and writes an `openapi.json`; no
  spec-to-client-code generator exists). A full automated scaffold-from-spec
  generator is still not unblocked, and remains out of this skill's own
  scope regardless — that's a separate, further step from
  `@zanix/console-kit` existing as an installable package.
- Further centralizing anything beyond what `@zanix/console-kit` already
  covers (the `AdminResource` shape, the hub-client-factory shape, the auth
  composition layer, column/detail-field derivation) into `@zanix/utils` or
  any other shared package. The kit itself is the answer to what this
  bullet used to defer — see this skill's own intro and layer 2's "Naming"
  note for the real trade-off centralizing it introduced. Don't go further
  than the kit's own current, extracted surface without a similarly real,
  confirmed second-consumer need for whatever's being proposed next (a
  generic page-shell factory was checked against exactly this bar and
  didn't clear it — see layer 4 above).
- Designing a `zanix new --template <name>` scaffold preset for this
  pattern. Real, already-built preset infrastructure exists in `@zanix/cli`
  (`ScaffoldRecipeRegistry`), but adding a second preset is a product
  decision to make only after this pattern has run manually against more
  than one real case — not an architecture question this skill settles.
- The backend-side API contract itself (RTOs, controllers, Discovery,
  `ServiceRegistry`) — that's `zanix-local-api-vs-aggregator`/
  `zanix-local-api-implementation`'s job. This skill only covers how a
  consumer builds UI against an already-existing remote contract.

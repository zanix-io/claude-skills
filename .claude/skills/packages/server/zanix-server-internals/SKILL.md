---
name: zanix-server-internals
description: Architectural invariants for extending @zanix/server itself — composition/activation layering, registration-function rules, the open core-slot registry, protocol design, and the auth-is-never-assumed rule. Use before designing or implementing a new cross-cutting mechanism, or before touching Application/Runtime/WebServerManager/route-registration code in this package. Not for application code that merely consumes @zanix/server (see zanix-server-conventions), and not for cross-package dependency/ownership questions (see zanix-dependency-direction and zanix-local-api-vs-aggregator).
---

This skill exists to stop the same architectural mistakes from being
reintroduced across sessions inside `@zanix/server` itself. Every rule below
was earned the hard way — either a design round caught it before it shipped, or
it shipped as a real bug that a later round found and fixed. File:line
references point at the actual precedent in
`~/Documents/Development/ZanixLibraries/server` (and the packages that consume
its registries) — read the real code there before assuming this summary is
still accurate; it will drift as the package evolves.

## Golden rule (token savings)

- Load only the section(s) that match what's actually being touched (the
  slot registry, the layering pattern, the typing design) — this skill covers
  several independent mechanisms; a change to one rarely needs re-reading the
  others.
- Verify a specific claim (a function's real signature, whether a class is
  actually exported from `mod.ts`) with a targeted grep/`deno doc` query, not
  by re-reading the whole file it lives in.
- The checklist at the end is the compact form of everything above — for a
  routine extension that clearly satisfies it, working through the checklist
  directly is enough; you don't need to re-narrate every section's reasoning
  to justify a "yes" answer.

## The layering pattern: composition → resolved plan → activation

Every cross-cutting mechanism in `@zanix/server` follows the same
three-ish-layer split. Don't build a new one (a new capability kind, a new
registry) without deliberately deciding where each piece of it lands in this
shape:

1. **Composition** — a capability declares what it needs, inside an ambient
   scope (`ProgramModule.defineApplication(name, setup)`), with zero knowledge
   of anything downstream. `RouteContainer.defineRoute`
   (`server/src/modules/program/metadata/routes.ts`) and
   `DiscoveryContainer.define` (`metadata/discovery.ts`) both just write plain
   metadata here — no compilation, no side effects.
2. **Resolved plan** — a *pure* function turns that metadata into a
   fully-specified, still-inert plan. `compileRuntime`
   (`modules/webserver/runtime.ts`) and `compileDiscoveryContract`/
   `compileHttpRuntime` (`modules/discovery/{provider,mount}.ts`) never touch
   `Deno.serve()` or any other real I/O — callable repeatedly with identical
   input for identical output, the same guarantee a good unit test should
   assert.
3. **Activation** — the *only* layer that touches real HTTP/sockets/side
   effects. `WebServerManager.create()` never derives
   Application/anchoring/Discovery semantics on its own — it only ever consumes
   an already-compiled plan. Keep it that way when extending it; the moment
   activation code starts asking "what Application is this" or "is this
   anchored," composition logic has leaked into the wrong layer.

**Activation-layer state that must survive a post-boot rebuild uses a frozen
box, swapped atomically — never mutated in place.** `WebServerManager`'s
`#handlers` (`modules/webserver/manager.ts`) is one `HandlerBox` per port,
each holding a `current` dispatch table built with `Object.freeze({...})`;
`create()` never mutates an existing table, it always builds a brand-new one
and reassigns `box.current`. `refreshRoutes(id)` (same file) is the reference
precedent for a mechanism that needs to recompile its own activation-time
output later, without rebinding the real `Deno.serve()` listener or losing
requests mid-swap: it recompiles the SAME box's `current` from
`ProgramModule`'s live registry and reassigns it in one atomic step, so
`multiplexer` (which dereferences `box.current` fresh per request, never
closing over a stale snapshot) always sees either the fully-old or
fully-new table, never a partial one. Any future mechanism whose own
activation output needs live-rebuild support (a dev-mode hot-reload for a
registry this package doesn't have yet) should copy this shape — freeze,
build fresh, atomic reassignment — rather than mutating the existing
structure or invalidating/rebinding the underlying listener.

**When a mechanism needs to support more than one transport** (Discovery: HTTP
now, an event bus later), split the resolved-plan layer itself into a
transport-agnostic piece (`DiscoveryContract` —
`resourceType`/`protocolVersion`, true regardless of transport) and a
transport-specific piece (`DiscoveryHttpRuntime` — `path`/`guards`, HTTP-only).
Don't let HTTP-only fields (a path, a header) leak into the transport-agnostic
layer just because HTTP is the only transport built so far — a later transport
built on the same foundation will need that boundary to already be clean.

**Lazy, programmatic route registration is a legitimate, precedented escape
hatch** — `RouteContainer.defineRoute`'s `applicationOverride` 3rd argument
exists specifically for capabilities that must register *after* composition has
finished, outside any `defineApplication` scope (GraphQL's own POST route,
`webserver/helpers/handler.ts`'s `getMainHandler`; Discovery's mount loop,
`webserver/mod.ts`'s REST branch). Never set this from decorator code — it's for
framework-internal lazy registration only, not something application code should
reach for.

## Registration functions: plain and re-callable, never a decorator or cached import

Confirmed empirically (a real Deno script, not assumed): a dynamic `import()`
of the same specifier only runs that module's top-level code **once per
process** — a second import returns the cached module namespace, no
re-execution. This is fine for a connector/provider registration that's meant
to survive forever once done (nothing purges `type:connector`/`type:provider`
target registrations mid-process — only `closeAllConnections()`, at real
process shutdown, does). It is **not** fine for anything whose backing registry
gets wiped between boot cycles — and the route registry
(`RouteContainer.resetContainer()`) and the discovery registry
(`DiscoveryContainer.resetContainer()`) both get wiped on every `finalize: true`
`bootstrapServers()` call
(`InternalProgram.cleanupInitializationsMetadata('postBoot', true)`,
`server/src/modules/program/mod.ts`).

**Rule**: anything that registers routes, resolvers, sockets, or discovery
providers must be a plain, exported, re-callable function — never a class
decorator relying on import-time evaluation, never wrapped in a cached
side-effect subpath import (`@some-package/core`-style). `@zanix/admin`'s
`defineAdminMetadata()` (`admin/src/modules/metadata.ts`) is the reference
example; its own doc comment states the exact reasoning. Before adding a new
`@some-package/core` subpath that does registration as a side effect, ask: does
this registry survive a finalized boot, or get wiped? If wiped, it needs a
callable function, not a cached import — this is not a style preference, it's a
correctness requirement the test suite (many independent `Zanix.bootstrap()`
cycles in one `deno test` process) will silently violate otherwise. Every
package that opts into the core-slot registry below (`auth`, `notifications`,
`asyncmq`, `datamaster`) follows this same rule from its own `/core`
entrypoint.

**A second, independent hazard the "plain re-callable function" rule doesn't
cover on its own**: this registration function still runs unconditionally at
module top level on first import (that's the whole point — it's what makes
the slot exist without a consumer calling anything). If the owning package
splits the registration call from the class/constant it reads across more
than one file, and those files end up importing each other, this exact idiom
is what produced a real, shipped crash in `@zanix/notifications`'s SMTP
connector (`email/defs.ts`'s `registerSmtpConnector()` reading `SmtpClient`
mid-cycle from `connector.ts`) — see `zanix-dependency-direction`'s
"intra-package circular imports with a top-level side effect" section for
the full precedent and checklist. Every package copying this skill's own
registration-function pattern inherits the same risk if its own file split
closes a cycle; check for one, don't assume the pattern is safe by itself.

## Open core-slot registry: `registerCoreProviderSlot`/`registerCoreConnectorSlot`

`@zanix/server`'s core capability slots are registered through an **open,
string-keyed registry** — not a hand-maintained union type
(`CoreProviders`/`CoreConnectors` in `typings/program.ts`) plus a hardcoded
`Record` (`providers/core/all.ts`, `connectors/core/all.ts`) statically wired in
`providers/core/mod.ts`/`connectors/core/mod.ts` that a new capability would
need to edit four files to extend. Modeled directly on
`DiscoveryContainer.define`'s own open, string-keyed registration (see the
layering section above) —
`registerCoreProviderSlot(key, BaseTarget, {sourcePackage?})`/
`registerCoreConnectorSlot(...)` (both exported from `@zanix/server`'s `mod.ts`)
let any package register a slot without `@zanix/server` enumerating it upfront,
from its own `/core` entrypoint. `CoreProviders`/`CoreConnectors`
(`typings/program.ts`) are now plain `string` — this registry is
**runtime-only**; see the typing section below for why it deliberately carries
no compile-time type information.

**Two separate registries, not one — pick the right reference example, don't
assume they're interchangeable.** `providerCoreModules`
(`providers/core/all.ts`) pre-seeds 5 slots: `cache`, `asyncmq`, `worker`,
`auth`, `notifications` — domain/orchestration capabilities, a layer *above*
a raw connection. `connectorCoreModules` (`connectors/core/all.ts`) pre-seeds
8: `cache:redis`, `cache:memcached`, `cache:custom`, `cache:local`, `kvLocal`,
`asyncmq`, `database`, `search` — raw connections to external infra. **These
lists are pre-seeded placeholders, not exhaustive** — any package can (and
does) register further slots at its own key from its own `/core` entrypoint,
outside this file's knowledge. A ninth, real, shipped connector slot already
exists and isn't listed above: `'s3'` (blob/object storage,
`datamaster/src/modules/storage/core.ts`,
`registerCoreConnectorSlot('s3', ZanixConnector, {sourcePackage:
'@zanix/datamaster/core'})`) — confirmed real, found only by grepping the
sibling repos directly, not by reading this file. A sixth, real, shipped
provider slot exists too and isn't listed above either: `'dlq'`
(`datamaster/src/modules/dlq/core.ts`,
`registerCoreProviderSlot('dlq', ZanixCoreDlqProvider, {sourcePackage:
'@zanix/datamaster/core'})`) — same shape as `auth`/`notifications`: no
dedicated `CoreBaseClass` getter, `this.providers.get('dlq')`/
`this.providers.get(DlqProvider)` is the permanent access pattern. **Before assuming no close
precedent exists for a new slot, grep
`registerCoreProviderSlot`/`registerCoreConnectorSlot` across the sibling
repos yourself** — this list drifts as packages evolve, exactly as this
skill's own intro already warns, and a close precedent (with its own
already-resolved getter/ownership reasoning) can save re-deriving a decision
from scratch.

**Which registry a new slot belongs to is a real decision, not
interchangeable — pick by what the capability actually is, not by copying
whichever example you read first.** A **connector** slot is a raw connection
to external infra (bytes in a database, bytes in a cache, bytes by key in
object storage) — same shape as `database`/`search`/`kvLocal`/`s3`. A
**provider** slot is domain/orchestration logic sitting a layer above a raw
connection (`cache` orchestrates over a `cache:*` connector; `auth`/
`notifications` are empty marker classes whose real logic lives entirely in
the owning package) — same shape as `auth`/`notifications`/`worker`.
`auth/src/modules/providers/core.ts` calling `registerCoreProviderSlot('auth',
ZanixCoreAuthProvider, {sourcePackage: '@zanix/auth/core'})` is a reference
example for a PROVIDER slot specifically — don't generalize it to "the
reference example for any new core slot"; a capability shaped like raw
external-infra access (a new storage backend, a new search engine, a new
queue transport with no domain logic on top) is a CONNECTOR slot, and
`datamaster`'s `'s3'` registration above is the closer precedent for that
shape.

**A subtle reachability gap worth knowing about**: an abstract contract class
that stays in `@zanix/server` (`ZanixAsyncMQProvider`, ...) can itself reference
its own slot key internally by string (`ZanixAsyncMQProvider.use()` calling
`this.getProviderConnector('asyncmq', ...)`). This bites internal test files
too: any test that decorates a class with
`@Connector({slot: 'cache:redis'})`/`@Provider({slot: 'cache'})` directly,
bypassing the owning package's `/core` entrypoint, must call
`registerCoreProviderSlot`/`registerCoreConnectorSlot` itself first — importing
`.../core/all.ts` alone doesn't run any package's registration side effect. This
applies to `@zanix/server`'s *own* functional test fixtures too, not just other
packages' tests — a fixture decorating `@Connector({slot: 'kvLocal'})` inside
`server`'s own test suite needs the same explicit registration call, since that
slot's registration lives in `@zanix/datamaster`, which `server`'s tests don't
depend on.

**Getter sugar (`this.cache`, `this.asyncmq`, ...) is a closed, deliberately
small list, gated by a concrete criterion — not an extensible mechanism, and not
an arbitrary one either.** `CoreBaseClass` (`modules/infra/base/core.ts`) only
grows a named getter for a slot when hosting that getter requires
`@zanix/server` to gain **no new domain knowledge** beyond a zero-behavior
abstract contract it would need to import anyway to write the getter's
return-type signature (e.g.
`T['cache'] extends ZanixCacheProvider ? T['cache'] : ZanixCacheProvider`).
That's true today for exactly 6 slots (`cache`, `database`, `asyncmq`, `worker`,
`kvLocal`, `search`) — a new package does **not** get to add its own `this.xxx`
by convention just because its capability feels important. `auth`/`notifications`
deliberately have no dedicated getter — `this.providers.get('auth')`/
`this.providers.get(AuthProviderClass)` is the correct, permanent access pattern
for them and for any future slot. Don't "fix" that asymmetry; it's the rule
working as intended. Before adding a new getter, ask: would this be the *first*
reason `@zanix/server` needs to import something about this domain? If yes, it
doesn't belong here — use `get('name')`/`get(Class)` instead.

**This same criterion decides where the slot's abstract contract class lives —
the two aren't independent rules.** A slot with a dedicated getter *must* keep
its contract in `@zanix/server`, because `CoreBaseClass`'s own getter signature
needs to import it directly — moving it would force
`server -> {datamaster,asyncmq}`, the exact reverse dependency
`zanix-dependency-direction` forbids. A slot *without* a dedicated getter has no
such constraint, and its contract belongs with the owning package from the
start. Concretely: `ZanixCoreAuthProvider` lives in `@zanix/auth`
(`auth/src/modules/providers/auth.ts`), `ZanixCoreNotificationsProvider` lives in
`@zanix/notifications` (`notifications/src/modules/providers/notifier.ts`) —
both are empty marker classes (`extends ZanixProvider<T> {}`, no body) whose
only job is to give `@Provider({ slot: 'auth' | 'notifications' })`'s
`instanceof` check something to validate against; `@zanix/server` never imports
either. Verify with `grep` before assuming a class like this "has to" live in
`server` just because the first 6 do.

## Class-based lookup resolves the same singleton as string-based lookup, for a core slot

`this.providers.get('cache')` and `this.providers.get(YourCacheClass)` — where
`YourCacheClass` is whatever concrete class was decorated
`@Provider({ slot: 'cache' })`, whether that's a package's own default
implementation or a consumer's rewrite — resolve the **identical cached
instance**, never two separate ones. Mechanism (`providers/core/all.ts`'s
`aliasCoreProviderTarget`/`resolveCoreProviderTargetAlias`, connector-side
mirror in `connectors/core/all.ts`): `defineProviderDecorator`/
`defineConnectorDecorator` (`.../decorators/assembly.ts`) alias `slot` back to
the slot's canonical string key. `getProviders`/`getConnectors`'s `get(Class)`
branch (`modules/program/public.ts`) checks this map *before* it ever reaches
`TargetContainer`'s instance cache, and resolves under the canonical string key
if an alias exists — so both lookup forms end up sharing the exact same cache
entry, never instantiating twice. A class never decorated for a core slot (the
common case — any custom provider/connector) has no alias and resolves under its
own class-derived key, unchanged.

**Never wrap a default core-slot implementation in a throwaway anonymous
subclass just to have a class-declaration site for the decorator — decorate the
real exported class directly, calling the decorator as a plain function, unless
the wrapper adds real behavior.** The alias mechanism above keys off
`getTargetKey(Target)` for whichever class was *actually* decorated — a pattern
like `class _X extends RealClass {}` followed by `@Provider('cache')` on `_X`
registers the *wrapper's* identity, not `RealClass`'s, so
`this.providers.get(RealClass)` (what every consumer actually imports and would
reach for) silently fails to resolve, even though `get('cache')` works fine.
This bit `auth`'s `ZanixAuthProvider`, `notifications`'s `NotifierProvider`,
`asyncmq`'s `ZanixCoreWorkerProvider`, and `datamaster`'s
`ZanixCacheCoreProvider`/`ZanixKVStoreConnector`/`ZanixQLRUConnector` — all
trivial `class _X extends RealClass {}` wrappers with zero specialization, fixed
by decorating `RealClass` directly (`Provider({slot:'cache'})(RealClass)`).
`@zanix/notifications`'s own `templates/core.ts` (`TemplateProvider`) already
did it the right way and had a regression test proving it. **The wrapper
pattern is still legitimate when it does real work**: `asyncmq`'s
`rabbitmq/defs.ts` wraps `ZanixRabbitMQConnector`/`ZanixCoreAsyncMQProvider`
specifically to inject `AMQP_URI` into the constructor — that one stays, since
there's no "real class" to decorate directly. The test before adding a wrapper:
does this subclass do anything besides exist for the decorator? If not, don't
wrap.

## `slot`, not `type` — the object-argument option on `@Provider`/`@Connector`

Renamed (breaking) because `type` never actually meant "kind of provider": for a
core slot it's literally the registration key (`key = slot` in `assembly.ts`),
but for a custom provider/connector it's discarded entirely after the decorator
runs — the real key comes from `getTargetKey(Target)`, and nothing downstream
ever reads the stored value back out (confirmed by grepping the whole monorepo
for `dataProps.type`/`.data.type` before renaming — zero hits). `slot` names
what the option actually does in both cases: which slot to register under, or
none at all. The single-argument string shorthand (`@Provider('cache')`) is
unaffected — only the object form's key changed. `dataProps: { slot,
autoInitialize }` (connector) was renamed to match, harmlessly, since nothing
reads it back either.

## Model/seeder registries are scoped per connector, via a `@zanix/server` primitive

`ProgramModule.models`/`ProgramModule.seeders`
(`datamaster/src/modules/program/metadata/{models,seeders}.ts`) are keyed by
database technology *and* connector instance — registering a second Mongo
connector doesn't wipe the first one's models, because both containers carry a
`connectorKey` dimension, resolved via a `@zanix/server` primitive rather than
requiring an explicit `slot`:

- `ZanixConnector.connectorKey` (`server/src/modules/infra/connectors/base.ts`)
  and the paired `getConnectorKey(Target)` (`server/src/utils/targets.ts`)
  expose the DI key a connector class was actually registered under —
  `'database'` for a class aliased to that core slot regardless of subclassing,
  or an auto-generated `getTargetKey`-derived key otherwise. This is what makes
  disambiguation automatic: two different connector classes always resolve to
  two different keys, whether or not either was given an explicit
  `@Connector({slot})`.
- `registerModel`'s second parameter (`database/defs/models.ts`) —
  `registerModel(model, connector?, type?)`, `connector` deliberately ordered
  before the rarely-used `type` — accepts the **connector class** directly (not
  a slot string), resolved via `getConnectorKey` at registration time, throwing
  immediately if the class hasn't been `@Connector`-decorated yet, rather than
  silently mismatching. Omitted, it targets `DEFAULT_CONNECTOR_KEY`
  (`'database'`), so a single-connector consumer sees zero behavior change.
- `ZanixMongoConnector`'s own `resolvedConnectorKey` getter
  (`mongo/connector/mod.ts`) falls back to `DEFAULT_CONNECTOR_KEY` whenever
  `this.connectorKey` is empty — needed for any connector instance that was
  never run through the real `@Connector` decorator (a test harness poking
  `_znx_props_` directly, a real, common case in `datamaster`'s own test suite).
  Without this fallback, such an instance's `connectorKey` is `undefined`, which
  never matches `registerModel`'s hardcoded default bucket.
- `getModel()`'s `ERR_MONGO_MODEL_NOT_FOUND` distinguishes "never registered"
  from "registered, but for a different connector" (`error.meta.kind`), naming
  the connector(s) it IS registered for in the latter case — via a permanent
  reverse index on `ModelsContainer` that survives `deleteModels`.

## `registerJob` deliberately does NOT take a `connector`/`provider` — jobs are transport-agnostic

Tempting to assume, by analogy with `registerModel`'s `connector` parameter
above, that `@zanix/asyncmq`'s `registerJob` should get the same treatment.
Considered and rejected — the two problems aren't actually analogous, and
copying the pattern would remove a capability the job system already has on
purpose.

`registerModel`'s `connector` param solves a **namespace-collision** problem:
multiple *simultaneous* Mongo connector instances all sharing one global
`ModelsContainer`, which needs a key to know whose bucket a given model belongs
to. `registerJob` (`asyncmq/src/modules/jobs/task.defs.ts`) has no such
problem — it writes to a single flat, global registry keyed only by job `name`
(`JOBS_METADATA_KEY`, `asyncmq/src/utils/constants.ts`), and there is currently
only ever **one** AsyncMQ broker slot in the whole framework (`'asyncmq'` — one
of `@zanix/server`'s 8 reserved core connector/provider slots).

More fundamentally: a job's registered handler is **dispatch-mechanism-agnostic
by design**. The exact same registered task (`TASKS_METADATA_KEY`, populated via
`registerTask`) runs identically through `ZanixCoreWorkerProvider.runJob`
(durable, via the AsyncMQ broker,
`asyncmq/src/modules/worker/provider.ts:135-165`) or `.runTask` (ephemeral, via
a local Web Worker thread, no broker/provider concept at all, same file:185-278)
— both converge on the identical `processor()`/`getTask(taskId)` resolution
(`asyncmq/src/modules/worker/queues/base.ts`). Baking a specific broker into a
job's *registration* would break the ability to run that same job either way
depending on the call site.

**Where the choice actually belongs, if/when it's ever needed**: `runJob`'s own
`options.provider` (`provider.ts:129`, defaults to `'asyncmq'`) — a
forward-looking, currently-inert option added specifically so a future second
AsyncMQ broker slot can be selected at the *dispatch* call site, never at
`registerJob`. The general lesson: before copying a `connector`/`provider`-
parameter pattern to a new registration function, check whether it's solving a
real namespace-collision problem (multiple simultaneous instances sharing one
registry) — if the thing being registered is actually transport-agnostic by
design, the parameter belongs on the *consumption*/dispatch call, not the
registration.

## Typing a string-keyed `get` call: explicit `CoreModules` generic, not ambient `declare module`

Making `CoreProviders`/`CoreConnectors` an ambiently-augmentable registry —
`keyof CoreProviderRegistry`, each owning package adding `declare module
'@zanix/server' { interface CoreProviderRegistry {...} }` next to its
`registerCoreProviderSlot` call — looks like the obvious way to give a
string-keyed slot a precise type. Don't build it: two problems rule it out.

1. **It would never actually give `this.providers.get('someKey')` a real
   per-key return type.** A `get` signature shaped like `<D>(Provider:
   ZanixProviderClass<D> | CoreProviders) => D` has a string branch that doesn't
   reference `D` anywhere, so passing a string literal never lets TypeScript
   infer a specific return type from it — `CoreProviders` only gates which
   strings are *legal arguments*, not what comes back. This holds even in pure
   local development with everything else working correctly — it isn't a
   JSR-specific gap.
2. **Ambient module augmentation isn't reliably visible from every file that
   needs it.** TypeScript only applies a `declare module` augmentation within
   files that are actually part of the same compiled program. `@zanix/server`'s
   own `mod.ts` doesn't import any sibling package's optional `/core` subpath,
   so a slot's type declaration living in the owning package leaves
   `@zanix/server`'s own `deno check mod.ts` broken wherever its abstract classes
   self-reference that slot by string. Separately, JSR's `no-slow-types`
   publish check explicitly flags any `declare module` block reachable via an
   `export * from ...` chain from a `deno.json` `"exports"` path — confirmed
   with `deno lint`, not just `deno check` (which doesn't run this rule at all).

**The design that works**: `CoreModules` (`typings/targets.ts`) is a plain
object type mapping a slot key to its type,
with the 6 foundational slots pre-declared as optional properties.
`ZanixInteractor<T>`, `ZanixProvider<T>`, `ZanixConnector<T>` (via
`CoreBaseClass<T extends CoreModules = object>`) all accept it as their one
generic parameter. `ZanixProvidersGetter<T>`/`ZanixConnectorsGetter<T>`
(`typings/targets.ts`) are `interface`s (not type-alias object literals —
interfaces support real method overloading) with three `get` overloads, tried in
this order: (1) a string key of `T` → returns `T[K]`'s declared type; (2) a
class reference → returns that class's instance type; (3) any other string →
the loosely-typed base provider/connector type. A consumer gets full typing for
a string key **only** by declaring it explicitly:

```ts
class MyInteractor extends ZanixInteractor<{ auth: ZanixCoreAuthProvider }> {
  method() {
    return this.providers.get("auth"); /* typed as ZanixCoreAuthProvider */
  }
}
```

There is no ambient/global fallback — an unparameterized class gets the
loosely-typed base class back for any string key outside the 6 foundational
ones, which is honest rather than falsely precise.

**Sharp edge, worth remembering if this pattern gets touched again**: the
overload-1 conditional must strip `undefined` before checking, i.e.
`NonNullable<T[K]> extends ZanixProviderGeneric ? NonNullable<T[K]> :
ZanixProviderGeneric` — not `T[K] extends ZanixProviderGeneric ? T[K] : ...`.
Every property on `CoreModules` is optional (`cache?: ZanixCacheProvider`), so
indexing an unmodified `CoreModules` key gives `ZanixCacheProvider | undefined`,
and `undefined` alone fails the naive `extends` check, silently falling through
to the generic fallback even when the key really is present. This surfaced as a
real bug (`ctx.providers.get('cache')` in `GuardContext` returning the generic
base type instead of `ZanixCacheProvider`), not something caught by design
review.

## Protocol consistency vs. ownership flexibility — two different axes

When introducing a shared protocol/envelope multiple resource kinds will speak
(Discovery's `resourceType`/`generatedAt`/`items` envelope, `versionProtocol`'s
header negotiation):

- **The protocol itself must be uniform and centrally enforced** — defined once,
  applied by the activation layer, never left for each resource's own
  registration code to reimplement. This is the entire value of having a
  protocol at all; letting each producer hand-roll its own envelope shape
  defeats it immediately.
- **Ownership of individual resources is allowed to differ freely** — don't
  force artificial symmetry (inventing a fake "owner module" for one resource
  just so it looks the same as another) to satisfy an aesthetic instinct. A
  protocol's job is to not care who's behind it; if it needs ownership symmetry
  to function, the protocol design is wrong, not the ownership.
- **Give each protocol its own version/header, even if the values start
  identical to another protocol's.** `DISCOVERY_PROTOCOL_HEADER`/
  `DISCOVERY_PROTOCOL_VERSION` (`server/src/utils/constants.ts`,
  `modules/discovery/provider.ts`) are deliberately distinct from
  `ADMIN_PROTOCOL_HEADER`/`ADMIN_PROTOCOL_VERSION`, even though both currently
  equal similar values — so one protocol's version bump never gets misread as
  the other's.

## Auth is never assumed — `@zanix/server` has no permissions concept of its own

`@zanix/server` depends on nothing that knows what a "role" or "permission" is —
that's `@zanix/auth`'s domain, and the dependency direction only ever goes one
way (see `zanix-dependency-direction`). Any new `@zanix/server` mechanism that
could plausibly need access control (Discovery, or a future one) must:

- Accept a plain guard slot (`MiddlewareGuard[]`) from the caller, exactly like
  any other route already does — never invent its own `permissions`/`roles`/
  `type` shape inside `@zanix/server`.
- Default to **no guard, not a fake one** — an empty `guards` array is the
  honest default; a registering module that forgets to pass one gets an
  unauthenticated endpoint, on purpose, not a silently-protected one that later
  turns out not to actually check anything.
- Reuse existing decorator-independent guard-producing functions instead of
  inventing new ones: `createProtocolVersionGuard`/
  `createProtocolVersionInterceptor`
  (`server/src/modules/infra/middlewares/protocol-version.ts`) and
  `@zanix/auth`'s own `jwtValidationGuard` (`auth/mod.ts`, the function
  `AuthTokenValidation`'s own decorator calls internally) are both plain
  functions callable outside any decorator — check for one of these before
  assuming you need to hand-roll middleware logic.

## Checklist before extending `@zanix/server`

- [ ] Does the new mechanism cleanly separate composition (ambient-scoped, no
      side effects) from resolved-plan (pure) from activation (the only layer
      touching real I/O)? If it needs more than one transport eventually, is the
      transport-agnostic part already split out?
- [ ] Does anything in it need to survive multiple boot cycles in one process
      (tests, or a real future multi-boot scenario)? If so, is registration a
      plain re-callable function — not a decorator, not a cached `/core`-style
      subpath import?
- [ ] Adding a new core provider/connector slot? Does the owning package call
      `registerCoreProviderSlot`/`registerCoreConnectorSlot` from its own
      `/core` entrypoint (no `declare module`/ambient typing needed — it can't
      give a string key a precise type and isn't reliably visible across
      files)? Does any internal test that bypasses
      the owning package's `/core` entrypoint call the registration function
      itself first? Will this new slot get a dedicated `CoreBaseClass` getter?
      If not (the default), its abstract contract class belongs in the
      *owning* package, not `@zanix/server`.
- [ ] Want a string-keyed `this.providers.get(key)`/`this.connectors.get(key)`
      call to return a precise type instead of the loose base class? Declare the
      key on the consuming class's own `CoreModules` generic
      (`ZanixInteractor<{ auth: ZanixCoreAuthProvider }>`) — never add a new
      ambient/global typing mechanism for this.
- [ ] If it's a shared protocol multiple resource kinds will speak, is the
      protocol itself uniform and centrally enforced, while ownership of
      individual resources stays free to differ? Does it have its own
      version/header, distinct from any other protocol's?
- [ ] Does it need access control? If so, does it accept a plain guard slot from
      the caller instead of inventing a permissions concept inside
      `@zanix/server` itself — and does omitting it produce an honest
      "unauthenticated," not a silently-broken "looks protected but isn't"?
- [ ] Has the actual behavior been verified with a real test/script, not just
      asserted from reading the code once?
- [ ] If the registration call and the class/constant it reads live in
      different files, do those files (and anything between them) form an
      import cycle? If so, does the registration call read a cross-cycle
      binding at its own top level — the shape that crashed
      `@zanix/notifications`'s SMTP connector (see the caution above)?

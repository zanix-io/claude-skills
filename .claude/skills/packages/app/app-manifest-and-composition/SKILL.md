---
name: app-manifest-and-composition
description: defineZanixApp() and the full manifest reference, AppContainer's pure half (normalize/buildGraph/validate, auto-bind), registerApp/ResourceRegistry/resolveResources (the runtime half), and activateApps/deactivateApps — the reference composition sequence. Use when authoring a Zanix App manifest, or composing/activating a set of apps.
---

Covers `@zanix/app`'s core authoring/composition surface. For `behaviors`/
`ctx.behavior()`/style-only overrides, see `app-behaviors-and-overrides`; for
Control Plane/`ctx.remote()`, see `app-remote-calls-and-control-plane`; for
leader election, see `app-leader-election-and-gateway`; for hot
install/multitenancy, MCP exposure, and sandboxing, see
`app-hot-install-and-multitenancy`, `app-mcp-composability`, and
`app-sandboxing`. File:line references point at
`~/Documents/Development/ZanixLibraries/app` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- Check the manifest reference table below before re-deriving a field's
  shape from prose — most fields are one row, not a concept needing
  re-explanation each time.
- Trust auto-bind for the common case (exactly one root resource of a
  slot's type) — only write an explicit `uses` binding when there's
  genuinely more than one candidate or none at all.

## Two layers — never confuse them

`@zanix/server`'s `ApplicationContainer`/`ProgramModule.defineApplication(name,
setup)` is the identity/routing/DI namespace primitive — it has no
manifest, no `dependencies`, no `resources`, no lifecycle concept of its
own. Every Zanix App IS an `Application` underneath (`registerApp` wraps
exactly one `ProgramModule.defineApplication` scope) — **but not every
`Application` is a Zanix App**. Reach for `defineZanixApp()` when a
dependency/resource/config/lifecycle manifest is the actual need; reach for
`defineApplication` directly only when none of that applies and a bare
namespace is genuinely all that's needed.

**Never call `AppContainer.registerApp`/`resolveResources`/`runOnStart`
individually from outside `@zanix/app` itself.** `activateApps` (or
`Zanix.start()`, which calls it) is the one correct sequence — calling
these out of order is a real risk (a resource resolved before its app is
registered, a job namespaced against the wrong scope), not a theoretical
one.

## Three entry points

```ts
import { ... } from '@zanix/app'          // pure manifest/types — zero dependency on @zanix/server
import { ... } from '@zanix/app/runtime'  // AppContainer/ResourceRegistry/ctx — depends on @zanix/server
import '@zanix/app/core'                  // side-effect only — zero-config Control Plane wiring
```

The split exists so anything that only needs to author or type-check a
manifest (a CLI scaffold, a build-time validator) never pulls in a full web
server. `@zanix/server` never imports from any of these — the dependency
graph is a one-directional DAG, not a cycle. `@zanix/app/core` is never
imported by `.`/`./runtime` themselves, so nothing pays for Control Plane
wiring unless a host explicitly opts in (see `app-remote-calls-and-control-plane`).

## `defineZanixApp()`

```ts
import { defineZanixApp } from '@zanix/app'

const reviews = defineZanixApp({
  name: 'reviews',
  routes: true, // auto-prefixes routes with "reviews"
  dependencies: { database: { type: 'mongo', required: true } },
  config: { apiKey: { type: 'string', secret: true, required: true } },
})

reviews.definition.name // 'reviews'
```

The only standard way to author a Zanix App — every field but `name` is
optional. Validates what it can **without any host context**: `name`'s
format (`^[a-z][a-z0-9-]*$`), and that a `secret: true` config entry never
carries a literal `default` (a secret must come from a host override or env
var, never hardcoded). Cross-app/host checks (`uses`/`dependencies`
contract, collisions) need the full dependency graph — see `AppContainer`
below.

**`.serve()`** — the author's own local dev loop, running THIS app alone,
in isolation (never what a real host does — a host installs via
`activateApps`, below):

```ts
const handle = await reviews.serve({
  resources: { mongo: { type: 'mongo', options: { uri: 'mongodb://localhost' } } },
  uses: [{ slot: 'database', resourceName: 'mongo' }],
  server: { rest: { port: 4000 } }, // omit to register without serving any HTTP surface at all
})
await handle.stop() // onStop → resources close → whatever server(s) serve() started
```

Implemented via a lazy `import('@zanix/app/runtime')` inside the method
itself — the exact same `activateApps`/`bootstrapAppServer`
`@zanix/core`'s own `Zanix.start()` calls, never a second, parallel
implementation. Importing `.` alone (never calling `.serve()`) still pulls
in zero `@zanix/server` dependency.

## Manifest reference

| Field | Shape | Notes |
| --- | --- | --- |
| `name` | `string` | Required. Route namespace + resource-key prefix + job prefix — must match `^[a-z][a-z0-9-]*$`. |
| `version` | `string?` | Stored only — no cross-app compatibility validation yet. |
| `runtime` | `{ mode?, replicas? }` | The author's own default execution-mode suggestion (`'embedded'` default, or `'remote'`) — never a command the app executes. `replicas` is a policy hint the Control Plane compares against what's observed (`app-leader-election-and-gateway`'s `compareReplicas`), not a number the app itself counts. |
| `routes` | `true \| false \| { prefix }` | `true` auto-prefixes with `name`; `false` registers no HTTP routes; `{ prefix: '' }` is an explicit opt-out of namespacing (distinct from `false` — the app still gets routes, just unprefixed). |
| `dependencies` | `Record<slot, { type, required? }>` | The closed, auditable set of resources this app can touch. Declares only the TYPE/shape needed, never a concrete resource name (that's the host's `uses`). |
| `config` | `Record<key, { type, default?, required?, secret? }>` | App-local parameters. `secret: true` never accepts a literal `default`. |
| `jobs` | `Record<name, JobDefinitionEntry>` | `JobDefinitionEntry` IS `@zanix/asyncmq`'s own `JobProcess` (`handler` + queue selection) plus optional `schedule`/`isActive`, referenced via `import type` — this shape can never drift from the real one. Namespaced internally to `${appName}:${jobName}`; `schedule` present routes to `registerCronJob`, absent to `registerJob`. |
| `operations` | `Record<name, OperationHandler>` | Named handlers OTHER Zanix Apps invoke via `ctx.remote(name).call(operationName, payload)` (`app-remote-calls-and-control-plane`). Separate from `routes` — never namespaced by path, never HTTP-shaped on the author's side. |
| `events` | `Record<name, {}>` | Declared, untyped payload for now. |
| `resources` | `Record<slot, {type, options} \| {type, mode:'remote', endpoint}>` | Local resources — shadows a root resource of the same name, only for slots also listed in `dependencies`. `mode: 'remote'` resolves to an RPC handle instead of a real instance (`app-remote-calls-and-control-plane`'s Remote Resource Binding). |
| `behaviors` | `Record<name, {default, description?}>` | Pure function/strategy slots a host can override — see `app-behaviors-and-overrides`. |
| `rootDir` | `string?` | Relative to the resolved package location (if `package` is set) or the host's cwd. |
| `package` | `string?` | Package specifier for a distributed app, loaded via `import(packageSpecifier)`. |
| `setup` | `(ctx: AppSetupContext) => void \| Promise<void>` | Programmatic registration escape hatch — `ctx.routes()`/`ctx.resolve()`/`ctx.resource()`/`ctx.config`. |
| `onStart` | `(ctx: AppStartContext) => void \| Promise<void>` | Runs sequentially, in declaration order across apps. |
| `onStop` | `(ctx: AppStopContext) => void \| Promise<void>` | Runs in parallel (`Promise.allSettled`) across apps. |

**Not yet implemented**: loading `rootDir`/`package` manifest files (an app
installed from disk/a package specifier); hot install/uninstall of
`jobs`/`events` (restart-only for now); an app catalog/marketplace.

## `AppContainer` (`.`) — composition, pure

```ts
import { buildGraph, validate } from '@zanix/app'

const graph = buildGraph(
  [reviews.definition],
  { mongo: { type: 'mongo', options: {} } },
  [{ appName: 'reviews', slot: 'database', resourceName: 'mongo' }],
)
validate(graph) // throws on a missing required dependency, a type mismatch, or an undeclared resource
```

**Auto-bind**: an explicit `uses` binding is only required when it's
actually ambiguous. If a slot has no binding at all, and exactly ONE root
resource's `type` matches `dependencies.<slot>.type`, `buildGraph` infers
the binding automatically:

```ts
const graph = buildGraph(
  [reviews.definition],
  { mongo: { type: 'mongo', options: {} } }, // the only 'mongo'-typed root resource
  [], // no uses.database binding at all — still resolves
)
```

Zero or more than one root resource of that type still requires an explicit
`uses`, fail-fast via `validate()`. Auto-bind is never considered when an
explicit (but broken) `uses` binding was given — a host that named
something that doesn't exist gets a real error, never a silent guess.

`normalize`/`buildGraph`/`validate` are the pure half of `AppContainer` — no
I/O, no `@zanix/server`. The other half (`resolveResources`, `registerApp`,
`runOnStart`/`runOnStop`) lives in `@zanix/app/runtime`.

## `registerApp` (`./runtime`)

```ts
import { registerApp } from '@zanix/app/runtime'
await registerApp(reviews.definition, resolvedResources)
```

Composes one normalized app into the running process: opens the app's own
`ProgramModule.defineApplication('reviews', ...)` scope, registers its
mount prefix (unless `routes: false`), registers every job namespaced to
`${appName}:${jobName}` (so two apps declaring a job of the same short name
never collide), and runs `setup(ctx)` if declared — `ctx.routes(register)`
runs registration inside this app's own scope, `ctx.resolve(Target)` sugars
over `ProgramModule.getInteractors`/`getProviders`/`getConnectors`
dispatched by which decorator `Target` extends, `ctx.resource(slot)`/
`ctx.config` are the same read-only accessors `onStart`/`onStop` get.

`resolvedResources` is the `Map<'${appName}:${slot}', instance>`
`resolveResources()` already produced — pass `new Map()` for an app with no
resources. Not this function's job: loading `rootDir`/`package` manifest
files, or producing `resources`/running `onStart`/`onStop` — those are
cross-app concerns `activateApps` owns.

## `ResourceRegistry` (`./runtime`)

Internal plumbing — an app's own code never calls this directly.

```ts
import { ResourceRegistry } from '@zanix/app/runtime'

const registry = new ResourceRegistry()
// Memoized by promise — a factory is invoked at most once per qualifiedKey,
// even if two callers ask for the same key concurrently before the first construction settles.
const db = await registry.resolve('mongo', () => connectToMongo())

// Promise.allSettled semantics — one resource's close() failing never stops the others;
// every failure is aggregated into a single AggregateError, never silently swallowed.
await registry.close()
```

## `resolveResources` + resource types (`./runtime`)

```ts
import { registerResourceType, resolveResources, ResourceRegistry } from '@zanix/app/runtime'

const resolved = await resolveResources(graph, registry)
resolved.get('reviews:database') // the real ZanixMongoConnector instance
```

`'mongo'`/`'redis'` are built in — `type: 'mongo'` resolves to
`@zanix/datamaster`'s real `ZanixMongoConnector`, referenced directly. A
host/package needing an unrecognized resource type registers its own
factory:

```ts
registerResourceType('my-custom-type', (options) => new MyOwnConnector(options))
```

Two apps bound to the same root resource get the exact same instance; an
app with its own local resource never shares it. If a constructed instance
is a real `ZanixConnector` (has both `isReady` and `isHealthy()`),
`resolveResources` health-gates it before resolving, reusing
`@zanix/server`'s own `connectorModuleInitialization` — resources built
here are constructed OUTSIDE the `@Connector`/`TargetContainer` path by
design, so `targetInitializations` never sees them otherwise. A plain
`CloseableResource` with no such concept (a custom factory, a test fake) is
never forced through this — it resolves as soon as its factory returns.

## `runOnStart`/`runOnStop` (`./runtime`)

```ts
import { runOnStart, runOnStop } from '@zanix/app/runtime'

await runOnStart([reviews.definition, billing.definition], resolvedResources) // sequential
await runOnStop([reviews.definition, billing.definition], resolvedResources)  // parallel (allSettled)
await registry.close() // resources are STILL OPEN after runOnStop — close only after it resolves
```

`runOnStart` is sequential, in declaration order, deliberately — two apps
sharing a resource could step on each other if their `onStart` ran
concurrently; determinism over speed for a boot sequence. `runOnStop` is
parallel and deliberately the opposite — the process is going down
regardless, so completing the most cleanup matters more than order, and one
app's `onStop` throwing never blocks another's.

## `activateApps`/`deactivateApps` (`./runtime`)

```ts
import { activateApps, deactivateApps } from '@zanix/app/runtime'

// normalize → buildGraph → validate → resolveResources → registerApp (sequentially, per app,
// each running its own setup(ctx)) → runOnStart (across every app)
const activated = await activateApps(
  [reviews, billing],
  { mongo: { type: 'mongo', options: { uri: 'mongodb://localhost' } } }, // root resources
  [{ appName: 'reviews', slot: 'database', resourceName: 'mongo' }], // uses bindings
)
activated.resources // the same Map every ctx.resource(slot) reads from
activated.registry // owns resources' construction/close lifecycle

// ... at shutdown — runOnStop (across every app) → registry.close(), even if an onStop failed:
await deactivateApps(activated)
```

`defs` accepts either the raw `AppDefinition` shape (normalized here) or
`defineZanixApp()`'s own return value directly (already normalized) — freely
mixed in the same call. Adds no logic of its own — every step is one of
this package's own already-tested primitives, called in the one order the
design requires. This is exactly what `@zanix/core`'s `Zanix.start()` calls
for any `apps.<name>` entry — never called directly from an app's own code.

## Checklist before authoring or composing a manifest

- [ ] Does `dependencies` declare only the TYPE/shape needed, never assuming
      a concrete resource name — that's the host's `uses` to decide?
- [ ] Is a `secret: true` config entry free of any literal `default`?
- [ ] Does auto-bind genuinely apply (exactly one root resource of that
      type), or does this slot need an explicit `uses` binding?
- [ ] Is a new field/mechanism actually the right one per the "Configuration
      vs Extension vs Override" table (`app-behaviors-and-overrides`) —
      `config` for one value, `resources` for something with a real
      lifecycle, `behaviors` for a pure function, a second app for genuinely
      new behavior?

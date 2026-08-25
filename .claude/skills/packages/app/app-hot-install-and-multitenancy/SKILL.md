---
name: app-hot-install-and-multitenancy
description: installApp/uninstallApp — the real sequence, the route-unmount gotcha (metadata removal never stops live traffic on its own), the jobs/events restart-only limitation, and ResourceRegistry's per-app quota/reference-counting that multi-tenancy isolation rides on (naming discipline, not an enforced primitive). Use when hot-installing/uninstalling an app into a running process, or isolating tenants by app name.
---

Covers runtime install/uninstall of one app into an already-running
process, and the multi-tenancy pattern built entirely on top of it. For the
initial batch composition (`activateApps`), see
`app-manifest-and-composition`. File:line references point at
`~/Documents/Development/ZanixLibraries/app` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- Check the jobs/events restart-only limitation before assuming a
  hot-installed app's scheduled work is live — it silently isn't, until a
  full process restart.
- Multi-tenancy here needs zero new mechanism if two tenants get distinct
  app names — don't reach for a `tenantId` concept that doesn't exist in
  this package.

## `installApp`/`uninstallApp`

```ts
import { installApp, uninstallApp } from '@zanix/app/runtime'

const activated = await installApp(activated, reviewsApp, {
  rootResources: { mongo: { type: 'mongo', options: {} } },
  bindings: [{ appName: 'reviews', slot: 'database', resourceName: 'mongo' }],
  maxResources: 5, // omit = unlimited; 0 = this app may never construct/reference any resource
})

const stillActivated = await uninstallApp(activated, 'reviews')
```

**Install sequence**: `buildGraph → validate` (the full merged graph, fail
fast before resolving/registering anything) `→ resolveResources` (delta
only, filtered to this app's own `${appName}:` keys) `→ registerApp →
runOnStart`. Throws `InternalError APP_ALREADY_INSTALLED` if `def.name` is
already active; throws from `validate()` if the contract is violated,
before touching resources; throws `InternalError
RESOURCE_QUOTA_EXCEEDED` if `maxResources` is set and exceeded.

**Uninstall sequence**: (1) fail-fast if another active app has a REQUIRED
`mode: 'remote'` dependency resolving to this app — throws `InternalError
APP_STILL_REQUIRED` with `meta: {appName, dependentApp, slot}` — (2)
deregister the Control Plane announcement +
`ProgramModule.unregisterApplicationRoutes(appName)` — (3) `runOnStop` —
(4) release every resolved resource via `ResourceRegistry.release` +
`closeSandboxedWorkers(appName)` (`app-sandboxing`). Throws `InternalError
APP_NOT_INSTALLED` if not active.

**Real footgun — uninstall failure leaves the app in an unknown state.** If
`onStop` throws (surfaces as `AggregateError`), routes/announcement are
already gone and resources are already released (in a `finally`), but
`onStop` may have only partially run — and the caller gets no updated
`ActivatedApps` back to reflect any of this.

**Real footgun — uninstall's dependency check only covers DECLARED
dependencies.** An ad-hoc `ctx.remote(appName)` call buried inside another
app's own operation/route handler has no manifest declaration —
uninstalling `appName` while such a call site exists just makes that call
start failing at its next invocation, with no warning at uninstall time.

## Real footgun: route metadata removal never stops live traffic by itself

`ProgramModule.unregisterApplicationRoutes` only removes metadata —
`@zanix/server`'s `getMainHandler` compiles its own route table once at
`WebServerManager.create()` time and never re-reads the registry
afterward. **The caller must separately call
`webServerManager.unmount(id)` for every `ServerID` the app owned** — the
framework doesn't track this for you. Skipping `unmount` means an
"uninstalled" app's routes keep dispatching real traffic. And even
`unmount` never closes the underlying socket, even when it was the last
server on a shared port — full teardown needs `stop()` on the port's
original owner too.

`unregisterApplicationRoutes` (metadata-only, keyed by Application name) is
the deliberately safer alternative to `@zanix/server`'s
`resetExceptApplications`, which requires enumerating every OTHER app to
preserve and silently wipes anything forgotten from that list.

## Real footgun: jobs/events are restart-only for hot install

`registerNamespacedJobs` runs and namespaces a hot-installed app's job
metadata correctly — but an already-running AsyncMQ worker/cron provider
snapshots its own metadata once at construction and never re-reads it. **A
hot-installed app's scheduled jobs silently never fire** until a full
process restart, even though installation itself reports success. Don't
assume `installApp` resolving means a new app's cron jobs are live.

## Multi-tenancy: a byproduct of naming discipline, not an enforced primitive

Every resolution key already scopes by app name — resource keys, config
override keys, route mount prefixes, `@zanix/auth`'s own rate-limit key.
Installing the same `defineZanixApp()` definition under a different `name`
per tenant already gives full isolation, with zero new mechanism. **There
is no `tenantId` as a first-class concept enforcing separation independent
of naming** — nothing in the framework stops installing two tenants under
the same or a colliding name; isolation depends entirely on the operator's
own naming discipline.

## `ResourceRegistry` quotas: reference counting, not resource limits per se

```ts
registry.setQuota('reviews', 5) // this app may reference at most 5 distinct qualifiedKeys
```

Enforced inside `resolve()`, **before** `factory` ever runs — a denied
request never pays construction cost. Counts DISTINCT `qualifiedKey`s
referenced by an app — referencing an already-shared root resource still
counts as one unit even though nothing new is constructed; re-resolving a
key already held never counts again. `uninstallApp` clears the quota
automatically (`clearQuota`), so a later reused app name never inherits a
stale ceiling.

**`maxResources: 0` is a valid, easy-to-misread value** — it means this
app may never construct/reference ANY resource (every `resolve()` call
rejects), distinct from omitting `maxResources` entirely (= unlimited).

**Real footgun — `resolve()` never retries a failed construction.** If the
memoized promise rejects, every caller (the original one and any that
arrived concurrently) gets that same rejection forever — only a fresh
`ResourceRegistry`/process retries. A transient construction failure isn't
self-healing.

Explicitly **not** built: CPU/memory/wall-clock sandboxing for resource
usage (see `app-sandboxing` for that, a separate mechanism entirely), or a
usage-billing/budget primitive ("N calls per month") — only a hard ceiling
on concurrent resource references.

## Checklist before hot-installing/uninstalling an app

- [ ] After uninstalling, is `webServerManager.unmount(id)` actually called
      for every `ServerID` the app owned — not just
      `unregisterApplicationRoutes`?
- [ ] Does this app declare any scheduled `jobs`? If so, is a full process
      restart planned after hot-install — hot-installed jobs don't fire on
      their own?
- [ ] Are two tenants given genuinely distinct `name`s — nothing else
      enforces the isolation?
- [ ] If `uninstallApp` might fail (`onStop` throwing), is the caller
      prepared for the app being left in an unknown, partially-torn-down
      state?

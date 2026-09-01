---
name: admin-service-registry
description: ServiceRegistry — the static known-services list TriggersAggregator and the templates sync endpoint both share via one installed instance, configured in code and/or ZANIX_ADMIN_SERVICES, plus checkServiceRegistryReachability for catching a stale adminBaseUrl before it fails deep in a real request, and RegistryHubClient for a remote caller reading the registry back over HTTP. Use when registering a service admin can reach, validating that registration, or building a remote UI/tool that lists what's registered on a hub.
---

Covers `ServiceRegistry`, the shared config both of admin's aggregating
mechanisms (`admin-triggers-aggregator`, `admin-templates-api`'s sync
endpoint) resolve services against. File:line references point at
`~/Documents/Development/ZanixLibraries/admin` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- Install one shared registry with `setServiceRegistry` before starting —
  don't let `TriggersAggregator` and the sync endpoint build independent
  instances that could drift out of sync.
- Run `checkServiceRegistryReachability()` (or `validateRegistry: true`)
  before assuming a registered `adminBaseUrl` is actually correct — a
  typo otherwise only surfaces the first time something real tries to use
  it.

## Configuring the registry — two supported paths

A **static** list — no dynamic self-registration. Both paths below are
confirmed exported from the package root (`jsr:@zanix/admin`) and can be
combined — an env entry overrides a constructor entry with the same
`serviceId`. Each entry's `serviceId` should match how that service is
known elsewhere: its own `ADMIN_SERVER_ID` and its registered
`JWK_PUB_<serviceId>` for authenticating to it (`admin-service-authentication`).

**Path 1 — in code (primary),** via `ServiceRegistry`/`setServiceRegistry`:

```ts
import { ServiceRegistry, setServiceRegistry } from 'jsr:@zanix/admin@[version]'

setServiceRegistry(new ServiceRegistry([
  { serviceId: 'billing', adminBaseUrl: 'http://billing.internal:30248/billing-rest' },
]))
```

**Path 2 — declaratively, via env var (no code change):**

```env
ZANIX_ADMIN_SERVICES=[{"serviceId":"billing","adminBaseUrl":"http://billing.internal:30248/billing-rest"}]
```

The env var's name is also exported as `SERVICE_REGISTRY_ENV`, so a
consumer that needs to reference it programmatically (tooling, deploy
scripts) doesn't have to hardcode the string:

```ts
import { SERVICE_REGISTRY_ENV } from 'jsr:@zanix/admin@[version]' // 'ZANIX_ADMIN_SERVICES'
```

## One shared registry, two consumers

`TriggersAggregator` (fanning out `/admin/triggers`/Discovery reads) and
the templates sync endpoint (pulling a service's `/.well-known/zanix/code-templates`
Discovery snapshot) both resolve the **same** installed instance via
`getServiceRegistry`, rather than each holding an independent one that
could drift. Install it once, before `ZanixAdminHub.start()`. Left
uninstalled, both consumers lazily build their own default instance the
first time either needs it (`ZANIX_ADMIN_SERVICES` entries only) — the same
instance from then on, since the first lazy build installs itself as the
shared one.

## Reachability validation

```ts
import { checkServiceRegistryReachability } from 'jsr:@zanix/admin@[version]'

const results = await checkServiceRegistryReachability()
// [{ serviceId: 'billing', adminBaseUrl: '...', status: 'ok', httpStatus: 400 }, ...]
```

A stale/typo'd `adminBaseUrl` is otherwise only discovered the first time
something actually tries to use it — `TriggersAggregator`'s and the sync
endpoint's calls have no try/catch around their own network hop, so a
config mistake surfaces as a production error deep in a real request, not
a clear signal at deploy time. `checkServiceRegistryReachability()` probes
every registered entry's `{adminBaseUrl}/admin/service-token` with an
intentionally-invalid credential — the same deliberately-safe-to-probe
route trust is established through
(`admin-service-authentication`) — and classifies each result:

| Status | Meaning |
| --- | --- |
| `'ok'` | A live, correctly-anchored admin server answered. |
| `'unreachable'` | A network error or timeout — nothing answered at all. |
| `'misconfigured'` | Something answered, but not this route (a 404) — a stale entry, wrong prefix, or wrong port. |
| `'unexpected'` | Any other response. |

Every per-entry failure is caught internally — this function can never
throw or hang its caller, and logs a warning for anything but `'ok'`.

```ts
await ZanixAdminHub.start({ validateRegistry: true }) // default false
```

`ZanixAdminHub.start()`'s `validateRegistry` option calls this
fire-and-forget, right after its servers are already listening, so a
temporarily-down registered peer never fails or delays boot. For a
deploy-pipeline smoke test instead (or alongside), script against the
returned array — exit non-zero if any entry isn't `'ok'`.

## Reading the registry remotely — `RegistryHubClient`

Everything above is the hub-server side (installing/validating the config
`ServiceRegistry` itself). A separate concern: a remote caller (e.g.
`@zanix/console`) that wants to READ what's registered on a running hub
does so over HTTP, via `RegistryHubClient` — a thin `RestClient` subclass
(`modules/registry/registry-hub.client.ts`, re-exported from this
package's own `mod.ts`) whose `list()` calls the hub's own
`GET /registry/list` route (`createRegistryController`, the same route
`ServiceRegistry`'s installed instance ultimately answers from). It
manages no credentials of its own — construct it with whatever the hub's
`AuthTokenValidation` accepts, same as every other client in this package.
Point its `baseUrl` at the HUB deployment itself, never at one of the
registered entries' own `adminBaseUrl` — those are a different level
entirely (a business service's own local admin API), confirmed by that
class's own doc.

## Checklist before registering a new service

- [ ] Does the new entry's `serviceId` match that service's own
      `ADMIN_SERVER_ID` and registered `JWK_PUB_<serviceId>`?
- [ ] Is the registry installed once via `setServiceRegistry`, before
      start, rather than left to two consumers building independent
      defaults?
- [ ] Has `checkServiceRegistryReachability()`/`validateRegistry: true`
      actually confirmed the new entry resolves, rather than assuming the
      URL is correct?

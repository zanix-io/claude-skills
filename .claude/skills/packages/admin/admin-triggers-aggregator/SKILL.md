---
name: admin-triggers-aggregator
description: TriggersAggregator's fan-out/proxy methods (list() via Discovery, CRUD proxied to the owning service), the pluggable clientFactory/discoveryClientFactory auth seam, TriggersController's HTTP surface, and createTriggersAdminController — the business-service-side local API this aggregator's client calls into (owned by @zanix/datamaster). Use when configuring trigger aggregation/proxying, or reviewing the split between the aggregator and a service's own local admin API.
---

Covers the Triggers side of `zanix-admin`'s dual API surface — the
aggregator/proxy, never the owner of any persisted trigger data. For the
Templates side, see `admin-templates-api`. File:line references point at
`~/Documents/Development/ZanixLibraries/admin` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- `ZanixAdminHub.start({ auth: { serviceId, privateKey } })` already wires
  an authenticated `TriggersAggregator` for the common case — hand-roll the
  factories below only when that shortcut doesn't cover a real need
  (custom `ServiceRegistry`, partial-failure tolerance, different
  credentials for CRUD vs. Discovery).
- Triggers stay owned per-service, always — `zanix-admin` never touches
  another service's database directly, only its `/admin/triggers` API.

## `TriggersAggregator`: fan-out/proxy over a `ServiceRegistry`

| Method | Behavior |
| --- | --- |
| `list()` | Fans out to **every** registered service's own `/.well-known/zanix/triggers` Discovery snapshot, tagged by `serviceId`. |
| `get(serviceId, model)` | Proxies to the one resolved service's CRUD API. |
| `create(serviceId, input)` | Proxies to the one resolved service's CRUD API. |
| `update(serviceId, model, changes)` | Proxies to the one resolved service's CRUD API. |
| `remove(serviceId, model)` | Proxies to the one resolved service's CRUD API. |

`list()` reads from Discovery rather than the CRUD API's own list route — a
read-only operation goes through the read-only protocol; every other
method mutates or targets a single entry, so it stays on the CRUD API.
**`TriggersAdminClient.list()` itself is real, working API surface — it's
just never called from inside `TriggersAggregator`.** It stays on the
client for a direct caller that genuinely wants one service's own list
without Discovery's involvement; confirmed deliberate precedent, not dead
code, the same shape any new aggregator mirroring this one (e.g. `DLQ`'s
own `DlqAdminClient.list()`/`DlqAggregator`) should also follow — don't
delete or "clean up" an admin client's own `list()` just because the
aggregator's own `list()` doesn't call it.

**Authentication is a pluggable seam.** The constructor takes two
independent factories — `clientFactory` (each per-service
`TriggersAdminClient`, CRUD) and `discoveryClientFactory` (each per-service
`DiscoveryAdminClient`, `list()`'s Discovery reads) — both default to
unauthenticated, which only works against a service that doesn't require
credentials for that surface. Both may return a `Promise`, since attaching
a real credential is inherently async.

```ts
const triggers = new TriggersAggregator(
  new ServiceRegistry([/* ... */]),
  async (service) => new TriggersAdminClient({ baseUrl: service.adminBaseUrl, headers: await authHeaders(service) }),
  async (service) => new DiscoveryAdminClient({ baseUrl: service.adminBaseUrl, headers: await authHeaders(service) }), // independent — could differ from the CRUD credential
)
```

A single service failing during `list()`'s fan-out fails the whole call
today (`Promise.all`) — deliberately simple; compose partial-failure
tolerance on top per `TriggersAggregator`'s own JSDoc if needed.

## `TriggersController`: the HTTP surface

| Route | Behavior |
| --- | --- |
| `GET /triggers` | `list()`. |
| `GET /triggers/:serviceId/:model` | `get(serviceId, model)`. |
| `POST /triggers/:serviceId` | `create(serviceId, body.model, body.active, body.triggers)`. |
| `PUT /triggers/:serviceId/:model` | `update(serviceId, model, {active?, triggers?})`. |
| `DELETE /triggers/:serviceId/:model` | `remove(serviceId, model)`. |

Install a real (authenticated) aggregator with `setTriggersAggregator`
**before** `ZanixAdminHub.start()` — left unset, the controller falls back
to a default `TriggersAggregator` (the shared registry, unauthenticated
clients) via `getTriggersAggregator`. Requires `ADMIN_ROLE`/
`ADMIN_TRIGGERS_ROLE` and accepts either a human admin's `type: 'user'`
token or a machine caller's `type: 'api'` one.

## `createTriggersAdminController`: the other side of the wire

Owned and authored by `@zanix/datamaster` (`@zanix/datamaster/triggers-api`)
— the actual owner of the triggers collection also owns the local HTTP
surface fronting it, per the "local API vs. aggregator API" rule
(`zanix-local-api-vs-aggregator`). **Don't confuse this with
`TriggersController`/`TriggersAggregator` above** — it's the other side of
the same wire protocol: a business service exposes this at a fixed
`admin/triggers` prefix, backed directly by
`TriggersAdminService`/`TriggersAdminRepository`, and it's what a
`TriggersAdminClient` calls into on the other end.

`@zanix/datamaster` never assumes an auth mechanism itself (no dependency
on `@zanix/auth`) — the controller factory accepts `guards`/
`versionProtocol` as options, supplied by whichever package composes it:

```ts
import { createTriggersAdminController } from 'jsr:@zanix/datamaster@[version]/triggers-api'
import { jwtValidationGuard } from 'jsr:@zanix/auth@[version]'

createTriggersAdminController({
  guards: [jwtValidationGuard({ permissions: [ADMIN_ROLE, ADMIN_TRIGGERS_ROLE], type: ['user', 'api'] })],
})
```

`@zanix/admin`'s own `defineAdminMetadata()` builds this exact guard — same
`ADMIN_ROLE`/`ADMIN_TRIGGERS_ROLE` gate and `type: 'user'`/`type: 'api'`
acceptance as `TriggersController` above; only the underlying business
logic differs (real CRUD vs. proxy). Registered under the `'admin'`
Application by default — `ADMIN_TRIGGERS_APPLICATION` overrides which
Application it's composed under instead.

## Checklist before touching trigger aggregation

- [ ] Is `zanix-admin` still treated as a pure proxy — never reading/writing
      a business service's own triggers collection directly?
- [ ] Is `setTriggersAggregator` (or `start({auth})`) installed before
      `ZanixAdminHub.start()`, so the controller isn't silently left on the
      unauthenticated default?
- [ ] For a new caller of `createTriggersAdminController`, does it build
      its own `guards`/`versionProtocol` — never assuming
      `@zanix/datamaster` provides a default auth mechanism?

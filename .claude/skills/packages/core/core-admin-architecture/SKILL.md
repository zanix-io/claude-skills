---
name: core-admin-architecture
description: This service's own embedded admin API vs. the centralized ZanixAdminHub — two independent HTTP servers, never two "modes" of one; the triggers-proxy-vs-templates-storage distinction; running both safely in one process; and TriggersAdminClient/TemplatesAdminClient for calling another service's admin API remotely. Use when deciding whether/how to run both this service's own admin API and ZanixAdminHub together.
---

Covers how `@zanix/core`'s own `admin` option relates to `@zanix/admin`'s
`ZanixAdminHub`. For the `admin` option's own configuration reference, see
`core-admin-apis`. File:line references point at
`~/Documents/Development/ZanixLibraries/core` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- These are two independent HTTP servers with independent route sets, not
  two modes of one API — don't reason about them as if enabling one
  changes what the other does.
- Register a service in `ZanixAdminHub`'s `ServiceRegistry` explicitly if
  you want the hub's triggers aggregator to include it — co-locating both
  in one process never wires them together automatically.

## Two independent servers, not two modes

```text
                         ┌─────────────────────────┐
                         │ ZanixAdminHub            │
                         │ /triggers   (proxy)      │
                         │ /templates  (own store)  │
                         │ /dlq        (proxy)      │
                         └─────────────────────────┘
                                      │
                        TriggersAdminClient over HTTP
                                      │
            ┬─────────────────────────┴┬──────────────────────────┬
            │                          │                          │
┌──────────────────────┐   ┌──────────────────────┐   ┌──────────────────────┐
│ billing               │   │ orders               │   │ inventory            │
│ /admin/triggers        │   │ /admin/triggers      │   │ /admin/triggers      │
│ /admin/templates       │   │ /admin/templates     │   │ /admin/templates     │
│ /admin/dlq             │   │ /admin/dlq           │   │ /admin/dlq           │
│ /admin/service-token   │   │ /admin/service-token │   │ /admin/service-token │
└──────────────────────┘   └──────────────────────┘   └──────────────────────┘
```

Each business service exposes its own internal admin API (`core-admin-apis`);
`ZanixAdminHub` is a separate application that may consume those APIs — it
never becomes them or replaces them.

## Triggers/DLQ: a pure proxy — templates: a storage decision

The relationship differs by domain:

- **Triggers**: `ZanixAdminHub`'s `/triggers` owns no trigger data of its
  own. `TriggersAggregator` fans `list()` out across every registered
  service via `DiscoveryAdminClient` (reading each service's own
  `/.well-known/zanix/triggers` snapshot), and proxies
  `get`/`create`/`update`/`remove` to whichever service the call names via
  `TriggersAdminClient`. This service's own `/admin/triggers` is the
  infrastructure the centralized aggregator depends on, not a redundant
  duplicate of it — a service only participates in aggregation if it
  exposes its own route, and disabling that module removes the service
  from what `ZanixAdminHub` can aggregate.
- **DLQ**: `ZanixAdminHub`'s `/dlq` mirrors the triggers shape exactly —
  `DlqAggregator` owns no DLQ data of its own and fans `list()` out across
  every registered service via `DiscoveryAdminClient` (reading each
  service's own `/.well-known/zanix/dlq` snapshot). Same proxy dependency
  as triggers: a service only shows up in the aggregated view if it
  exposes its own `/admin/dlq` route.
- **Templates**: `ZanixAdminHub`'s `/templates` owns its own data — its own
  `TemplatesAdminService` and database connector, entirely independent of
  any business service's local one. This makes templates a **storage
  choice**, not an aggregation dependency — pick one of two deployment
  models (per-service storage with a local `/admin/templates`, or
  centralized storage with `RemoteTemplateBackend` pulling from the hub).
  Running both at once means two genuinely independent template stores,
  not one synced across two APIs, unless the app explicitly keeps them in
  sync.

## Running both servers safely

```ts
import Zanix, { ZanixAdminHub } from 'jsr:@zanix/core@[version]'

// Neither call needs to be awaited before the other starts — each runs under its own isolated
// boot session, so registering/booting them concurrently is safe.
const core = Zanix.start({ admin: true }) // this service's own business API + local admin API
const hub = ZanixAdminHub.start({ rest: { port: 3001 } }) // the centralized admin hub, in the same process
await Promise.all([core, hub])
```

Running `Zanix.start()` (admin at its default `false`) and
`ZanixAdminHub.start()` in the same process starts two completely
independent HTTP servers — co-locating them doesn't connect them
automatically. To have `ZanixAdminHub`'s triggers aggregator include this
service, register this service's own `ADMIN_SERVER_ID` address in the
hub's `ServiceRegistry` (`ZANIX_ADMIN_SERVICES`) explicitly.

**Enabling `admin` on `Zanix.start()` in the same process as
`ZanixAdminHub.start()` is supported and safe**, including calling both
back to back with no `await` between them
(`Zanix.start({admin:true}); ZanixAdminHub.start()`) — each independently
registers its own metadata and calls `bootstrapServers()`, and
`@zanix/server`'s cleanup is scoped to each top-level sequence's own boot
session (`BootSessionContainer`), so two independent sequences can never
wipe each other's not-yet-served routes. The two route sets also compose
under distinct Applications (`'admin'` for this service's own embedded
API, `'admin-hub'` for `ZanixAdminHub`'s own aggregator), keeping them
logically distinguishable on top of that. `admin: true` is only needed if
this service should expose its own local admin API alongside hosting the
hub — the common "business API + a separate hub, nothing local"
combination only needs `Zanix.start()`'s default.

**Real gotcha**: sharing one port between `Zanix.start()`'s public server and
`ZanixAdminHub.start()` (or its embedded `admin` server) is supported by
design, not a failure mode — `WebServerManager` binds one real `Deno.serve()`
listener per port and dispatches every server sharing it by a `dispatchKey`
lookup table (`@zanix/server`'s `manager.ts`, `create()`'s own "Sharing a
port" remarks and `shared-port.test.ts`), so this never throws `AddrInUse`.
The actual risk is a **silent dispatch-key collision**: if two servers on the
same port ever resolve to the identical dispatch key — the same anchored id,
or the same `globalPrefix` when unanchored — the later `create()` call's
handler replaces the earlier one's for that key with no error, silently
making the first server's routes unreachable (`manager.ts`'s own `box.current
= Object.freeze({...box.current, [dispatchKey]: dispatchHandler})`). This
doesn't happen by default: the public server's own default prefix (`'api'`),
the embedded local admin's own unanchored default (`` `admin-${type}` ``),
and `ZanixAdminHub`'s own unanchored default (`'admin-hub'`) are all
distinct, which is exactly why the hub defaults to `'admin-hub'` in the first
place — see `admin-hub-single-port-unanchored.test.ts`, which asserts all
three sharing one port with zero anchoring config. It can still happen if you
explicitly set matching `globalPrefix`es on two of them, or reuse the same
value for both `ADMIN_SERVER_ID` and `ADMIN_HUB_SERVER_ID` — those are
separate env vars specifically so the embedded admin and the hub can each be
anchored without colliding with each other.

## Why `/admin/service-token` is always registered

It isn't scoped to either domain — it's the generic machine-to-machine
authentication primitive any caller needs before calling anything gated by
a `type: 'api'` token, whether that's `TriggersAdminClient`,
`TemplatesAdminClient`, `ZanixAdminHub`'s own aggregator,
`RemoteTemplateBackend`, or a future caller. That's why it stays available
even when neither `/admin/triggers` nor `/admin/templates` is registered.
Its presence alone grants nothing — the endpoint only exchanges a valid
service assertion for an API token, and rejects any caller whose
`JWK_PUB_<serviceId>` isn't registered.

## Calling another service's admin API remotely

```ts
import { TriggersAdminClient } from 'jsr:@zanix/core@[version]'

const client = new TriggersAdminClient({
  baseUrl: 'http://billing.internal:8000/billing-rest', // that service's port + ADMIN_SERVER_ID prefix
  headers: { 'X-Znx-Authorization': `Bearer ${accessToken}` },
})
const triggers = await client.list()
```

`@zanix/admin` owns the request/response contract's client-side
counterpart (re-exported here for convenience) — a consumer never
hand-rolls its own HTTP client for this. `TemplatesAdminClient` mirrors the
same shape for `/admin/templates`. Both never send `updatedBy` — the
target service infers it from the caller's own authenticated session, same
as the local API already does.

## Checklist before running this service's admin API alongside `ZanixAdminHub`

- [ ] Is this service's `ADMIN_SERVER_ID` actually registered in the hub's
      `ServiceRegistry` if the hub's triggers aggregator needs to include
      it — co-location alone never wires them together?
- [ ] If both servers share the same port in the same process, do they
      resolve to genuinely distinct dispatch keys — different anchored ids
      (`ADMIN_SERVER_ID` vs. `ADMIN_HUB_SERVER_ID`, never the same value for
      both) or different `globalPrefix`es — rather than relying on an
      unverified assumption that the unanchored defaults never collide?
- [ ] Is `admin: true` genuinely needed here — or does this deployment only
      need the common "business API + separate hub" combination, which
      needs no `admin` option at all?
- [ ] For templates specifically, is the per-service-vs-centralized storage
      choice deliberate, with both stores' independence (or explicit
      sync strategy) understood?

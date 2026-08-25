---
name: app-remote-calls-and-control-plane
description: The Control Plane (Redis-backed app registry + hot-refresh config), ctx.remote()'s local-first-then-HTTP resolution, allowedCallers app-to-app ACLs (enforced at three dispatch points), Remote Resource Binding, distributed lifecycle announcement, and mTLS (outgoing and the real Deno TLS-verification gap on incoming). Use when an app calls another app's operation, or exposes one to be called remotely.
---

Covers how Zanix Apps call each other across process boundaries. For leader
election and the Gateway, see `app-leader-election-and-gateway`; for the
manifest/composition layer this builds on, see
`app-manifest-and-composition`. File:line references point at
`~/Documents/Development/ZanixLibraries/app` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- Trust `ctx.remote()`'s local-first resolution — it costs zero network
  when the target app is active in the same process, and needs no special
  handling at the call site either way.
- `allowedCallers` is opt-in, not secure-by-default — check whether an
  operation actually needs it before assuming a bare handler function is
  already protected.

## Control Plane: registry + config, both Redis-backed

```ts
import { ControlPlaneConfig, ControlPlaneRegistry } from '@zanix/app/runtime'
import { ZanixRedisConnector } from '@zanix/datamaster'

const registry = new ControlPlaneRegistry(new ZanixRedisConnector())
const configPlane = new ControlPlaneConfig(new ZanixRedisConnector())
```

Both constructors take an **already-constructed** `ZanixCacheConnector<'redis'>`
— neither class builds its own connection. `ZanixControlPlaneProvider`
(registered under the `'controlPlane'` core-provider key by `@zanix/app/core`)
reuses `this.cache.redis`, shared with `@zanix/datamaster/core`'s own
`'cache'` provider — never a second connection.

- **`ControlPlaneRegistry`**: `registerInstance(appName, instanceId,
  {prefix, endpoint}, {leaseTtlSeconds})` — one Redis key per replica,
  TTL-scoped; the same call re-registers/renews (no separate `renew()`).
  `getDeploymentTarget(appName)` returns `{mode: 'remote', prefix,
  endpoints}` or `undefined` (no live replica) — never throws.
  `deregisterInstance` is graceful/best-effort.
- **`ControlPlaneConfig`**: `setConfig(appName, key, value)` writes and
  publishes to subscribers; `getConfig` is a cold read;
  `subscribeConfig(appName, keys, callback)` returns a subscription with
  `.close()` (releases its dedicated Pub/Sub connection).

**Caution — a `secret: true` config key must never go through
`ControlPlaneConfig`.** Secret config is never subscribed for hot-refresh —
enforced by the function itself, not left to caller discipline.

## `ctx.remote()`: local-first, HTTP fallback

```ts
await ctx.remote('billing').call('chargeInvoice', { orderId }, { timeoutMs: 3000 })
```

Available in `setup`/`onStart`/`onStop`, and inside `operations` handlers.
`timeoutMs` is mandatory, no default. Resolution order: (1) local-first,
zero network, if the target app registered the operation in this same
process; (2) HTTP fallback via `HttpRemoteAdapter`, only if (1) finds
nothing. Enable (2) via the side-effect import `'@zanix/app/core'`, which
registers the `'controlPlane'` slot that `activateApps` auto-detects —
without it, `ctx.remote()` stays local-only at zero cost. A manual
`dispatcher` (`activateApps`'s 4th arg) always wins over the auto-detected
provider.

`HttpRemoteAdapter` resolves the target's live endpoint via the Control
Plane Registry, authenticates via `@zanix/auth`'s
`createServiceAuthClient`/`exchangeServiceCredential` (same mechanism
`ZanixAdminHub` already uses), propagates a W3C `traceparent`, and enforces
`timeoutMs` via `AbortSignal.timeout()`. Errors are always a real
`InternalError` — `REMOTE_APP_UNREACHABLE` (no live instance),
`REMOTE_CALL_TIMEOUT`, `REMOTE_CALL_FAILED` (other transport/HTTP failure)
— **never a silent `undefined`**.

**Wire protocol**: a remotely-callable app is served under the fixed path
`/__zanix-ops/${appName}/...`, independent of the app's own
`routes`/mount prefix — `POST .../service-token` for auth exchange, `POST
.../:operationName` protected by `@AuthTokenValidation({type: 'api'})`.
Registered automatically whenever an app declares `operations`; a no-op
(zero routes) otherwise.

## `allowedCallers`: app-to-app authorization, opt-in

```ts
operations: {
  listInvoices: async (payload, ctx) => ({ invoices: [] }), // bare fn = fully public, unchanged
  refundOrder: {
    handler: async (payload, ctx) => ({ refunded: true }),
    allowedCallers: ['reviews'], // '*' or omitted = public
  },
}
```

Checked against the **calling app's own identity** — `ctx.remote()`'s
`callerAppName` (same-process) or the exchanged service token's `sub` claim
(cross-process). Never a human/end-user identity — that's `@zanix/auth`'s
separate `@RequirePermissions` on `routes`, a different concern entirely.
Enforced at **three** dispatch points: local-first (`remote-caller.ts`),
remote HTTP (`remote-dispatch-route.ts`'s `dispatch()`, right after
`@AuthTokenValidation` — denied with `HttpError('FORBIDDEN')`), remote mTLS
(`mtls-dispatch-server.ts`'s `handleRequest()`, right after the service
token's `sub` is verified — denied with `403`).

**Deliberately opt-in, not secure-by-default**: a bare function or an
omitted `allowedCallers` is fully callable, exactly as before this feature
existed. Check whether a genuinely sensitive operation needs it — the
absence of `allowedCallers` is never itself a signal that an operation was
reviewed and found safe to leave open. This is app-to-app authz only — it
does not thread through which human/end-user triggered the original call;
there's no end-user scope propagation today.

## Remote Resource Binding

```ts
const reviews = defineZanixApp({
  name: 'reviews',
  dependencies: { database: { type: 'mongo', required: true } },
  resources: { database: { type: 'mongo', mode: 'remote', endpoint: 'billing' } },
})
```

**Deliberately not transparent** — a local `database` resolves to a real
connector (`.find()`/`.insertOne()`), a remote one resolves to a
`RemoteAppHandle`, the exact same shape `ctx.remote(endpoint)` returns:

```ts
const db = ctx.resource('database') // RemoteAppHandle, not a Mongo connector
const result = await db.call('findAccount', { id: 42 }, { timeoutMs: 3000 })
```

True transparency was rejected — it would need either a hand-written proxy
per resource type or blanket reflection-based forwarding, both new
mechanisms this package would then own and maintain. Reuses `ctx.remote()`'s
exact contract (same local-first resolution, same
`HttpRemoteAdapter`/service-token/`traceparent` otherwise, same
`REMOTE_APP_NOT_CONFIGURED` error if neither applies). `type` is still
checked against `dependencies.<slot>.type` by `validate()`, same as a local
resource.

**`requiredVersion`** (`{..., requiredVersion: '^1.0.0'}`, `@std/semver`
format): checked by `validate()` only when it genuinely can be — only if
`endpoint` is part of the SAME composition (same `apps` list passed to that
`validate()` call) and declared its own `version`. A genuinely cross-process
remote target is **not checked at all** — an honest, documented limitation,
since `validate()` is deliberately synchronous/fail-fast and doesn't do
async Control Plane lookups. Errors: `REMOTE_RESOURCE_VERSION_MISMATCH`
(checked and unsatisfied), `INVALID_VERSION_RANGE`.

## Distributed lifecycle: `activateApps`'s `remoteInstances`

```ts
await activateApps(defs, rootResources, bindings, dispatcher, {
  reviews: { endpoint: 'https://reviews-a.internal:8443', leaseTtlSeconds: 30 },
})
```

An app listed in `remoteInstances` announces itself to the Control Plane
**after** its own local `onStart` already completed — registers in the
Registry, renews its own lease on a heartbeat (default: a third of
`leaseTtlSeconds`), and subscribes to hot-refresh for every non-secret
declared config key. `deactivateApps` tears down symmetrically: deregister
first (best-effort, so a Gateway stops routing to it) → the app's own
`onStop` runs → resources close.

**Presence in `remoteInstances` IS the host's decision** to run an app in
`remote` mode for this process — the manifest's own `runtime.mode` is only
a default suggestion, never enforced by itself. An app not listed is never
announced regardless of what its manifest says.

## mTLS: outgoing is easy, incoming needs a dedicated listener

**Outgoing** (`HttpRemoteAdapter`'s constructor 2nd arg):

```ts
new HttpRemoteAdapter(registry, {
  cert: await Deno.readTextFile('./client.pem'),
  key: await Deno.readTextFile('./client.key'),
  caCerts: [await Deno.readTextFile('./ca.pem')], // only if target's server cert isn't Deno-trusted
})
```

The certificate covers the whole round trip (service-token exchange and the
operation call). `dispatcher.close()` releases the TLS connection pool.

**Real platform gap, worth knowing before assuming this is symmetric**:
current stable Deno's `Deno.serve()`/`Deno.listenTls()` have no mechanism
to require or verify an incoming client certificate
([denoland/deno#26825](https://github.com/denoland/deno/issues/26825)) — a
regular app's `routes` cannot reject an uncertified caller at the TLS layer.

**Incoming (opt-in)** — `/__zanix-ops/...` uses a **separate**, dedicated
listener built on Deno's `node:https` compat layer, which genuinely
enforces `requestCert`/`rejectUnauthorized`:

```ts
import { announceRemoteInstance } from '@zanix/app/runtime'

const announced = await announceRemoteInstance(reviews.definition, {
  endpoint: 'https://reviews-a.internal:8443',
  mtls: {
    port: 8443,
    cert: await Deno.readTextFile('./server.pem'),
    key: await Deno.readTextFile('./server.key'),
    ca: [await Deno.readTextFile('./ca.pem')], // connecting client's cert must chain here
  },
}, registry)
// announced.stop() closes the mTLS listener too, alongside heartbeat/deregistration.
```

Deliberately narrow — this second listener serves ONLY
`/__zanix-ops/${appName}/...`, never touching `@zanix/server`'s
`webServerManager`/`Deno.serve()` routing. **mTLS adds transport-layer
identity on top of the application-layer service-token exchange, never a
replacement for it** — `@AuthTokenValidation({type: 'api'})` still gates
access independently even when mTLS is configured.

## Checklist before adding/reviewing a cross-app call

- [ ] Does a genuinely sensitive operation set `allowedCallers`, rather than
      assuming an unlisted default is already protected?
- [ ] Does the call site pass a real `timeoutMs`, not an arbitrarily large
      one that defeats the point of having a mandatory timeout?
- [ ] For Remote Resource Binding, is `requiredVersion` set with the
      understanding that it's only checked within the same composition —
      never for a genuinely cross-process target?
- [ ] If incoming mTLS is configured, is it understood as additive to
      `@AuthTokenValidation`, never a substitute that would let a route skip
      application-layer auth?

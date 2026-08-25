---
name: app-leader-election-and-gateway
description: Automatic Redis-backed leader election for scheduled jobs (fencing tokens, the Redlock multi-connector upgrade), compareReplicas as a pure diagnostic, and the Gateway's by-name-then-by-default routing for public traffic. Use when a scheduled job needs exactly-once-per-tick execution across replicas, or when routing external traffic to a remote app.
---

Covers multi-replica coordination and public request routing. For
Control Plane/`ctx.remote()`/mTLS, see `app-remote-calls-and-control-plane`.
File:line references point at `~/Documents/Development/ZanixLibraries/app`
— read the real code there before assuming this summary is still accurate.

## Golden rule (token savings)

- Scheduled jobs get leader election automatically — no manifest opt-in
  needed. Only add `isJobFencingTokenCurrent` when a job commits a real side
  effect that must never double-apply, not for every job by default.
- `compareReplicas` is a pure diagnostic, never enforcement — don't treat a
  mismatch as something the package itself will correct.

## Leader election for scheduled jobs

A `jobs.<name>` entry with `schedule` is automatically wrapped with
Redis-backed leader election via `registerNamespacedJobs` — only the
replica holding `${appName}:${jobName}`'s lease runs the handler; every
other replica no-ops that tick. The lease renews every tick; on a crash or
partition, the next tick's live replica acquires it fresh once the TTL
lapses. Mechanism: a real atomic `SET NX EX` plus a Lua compare-and-extend
script (`LeaderElection`), via the same `getClient()` escape hatch
`ZanixKVConnector.withLock()` documents.

**Not the same mechanism as `@zanix/asyncmq`'s `lockMessage`** — that one is
a real mechanism too, but a non-atomic check-then-set that only *reduces*,
not eliminates, duplicate execution under real concurrent delivery. Leader
election here is atomic and eliminates it for the tick it protects.

## Fencing tokens: eliminating double-effect, not double-dispatch

```ts
import { isJobFencingTokenCurrent } from '@zanix/app/runtime'

jobs: {
  chargeInvoices: {
    schedule: '0 0 * * * *',
    processingQueue: 'soft',
    handler: async function (args) {
      if (!(await isJobFencingTokenCurrent('billing', 'chargeInvoices', this.context))) {
        return // a newer leadership term already started — stale run, skip commit
      }
      // ... commit the side effect ...
    },
  },
}
```

`getJobFencingToken(this.context)` returns the raw token if something
beyond the built-in comparison is needed. **Explicit, real limitation**:
this doesn't eliminate the double-*dispatch* window — a network partition
under a TTL-based lease is an inherent limit of the approach, not something
specific to this package or Redis. What it does eliminate is
double-*effect*: a stale replica that still fires can check the token and
skip committing, even if it briefly believed itself leader.

## `compareReplicas`: a pure diagnostic, never enforcement

```ts
const { declared, observed, matches } = await compareReplicas(reviews.definition, registry)
// declared: def.runtime.replicas (null if unset)
// observed: live instance count
// matches: true if nothing to compare, or declared === observed
```

Read-only — it never scales anything up or down, never corrects a mismatch.
Use it for alerting/dashboards, not as a control mechanism.

## Redlock: scaling leader election across multiple Redis instances

```ts
import { LeaderElection } from '@zanix/app/runtime'
const election = new LeaderElection([connectorA, connectorB, connectorC])
```

A single connector (the default) behaves as single-instance leader
election. An array of connectors upgrades to majority-quorum Redlock, same
public API either way (`tryAcquireOrRenew`/`getCurrentFencingToken`/
`release`). Tolerates a minority being unreachable — succeeds once a
majority (`floor(N/2)+1`) agree, applying Redlock's clock-drift discount
(nominal TTL minus round trip minus drift allowance). Per-instance
operations are internally bounded to a short timeout, so one unresponsive
(not rejecting, just hanging) instance can't block the whole quorum check.

**Real footgun**: never point more than one of these connectors at the same
physical Redis instance — that buys no real fault tolerance, it just
repeats the same single point of failure under a different name. Redlock's
guarantees depend on the instances genuinely being independent.

## Gateway: routing public traffic to a remote app

```ts
import { bootstrapAppServer, createGatewayPreHandler } from '@zanix/app/runtime'

const preHandler = createGatewayPreHandler(registry, {
  localAppNames: ['this-process-own-app'],
  defaultRemoteApp: 'storefront',
})

await bootstrapAppServer('this-process-own-app', { rest: { port: 8080, preHandler } }, true)
```

Built on `@zanix/server`'s `preHandler` extension point (the same one
`@zanix/space`'s dev server uses) — tried before route matching on every
request; returning `undefined` falls through to normal routing unchanged,
so it costs nothing beyond that check for non-Gateway traffic.

**Resolution order**:
1. **By name** — the request path's first segment is looked up directly in
   the Control Plane Registry. Requires the remote app's served routes to
   literally start with its own `name` (`billing` at `/billing/...`, never
   `/api/billing/...` — no REST default `/api` prefix, no anchored server
   `id`).
2. **By default** (`defaultRemoteApp`) — tried only if (1) finds nothing;
   needed for whole-domain apps (e.g. `@zanix/space` with `routes: {prefix:
   ''}`) where the path carries no app-identifying segment. **Only one
   default makes sense per Gateway** — two whole-domain apps sharing an
   origin would collide.

`localAppNames` is checked **before** either strategy — a locally-mounted
app is never shadowed by remote routing. Pass `defs.map(d => d.name)`, the
same defs given to `activateApps`.

The reverse proxy forwards method/headers/body as-is, **streamed, never
buffered**, round-robin per resolved app name via `RoundRobinPicker` (the
same mechanism `HttpRemoteAdapter.dispatch` uses). An unreachable target
returns `502` directly — the Gateway itself never throws.

## Checklist before adding a scheduled job or Gateway route

- [ ] Does a job that commits an irreversible side effect check
      `isJobFencingTokenCurrent` before committing — not just rely on
      leader election alone?
- [ ] If upgrading to Redlock, are all the connectors genuinely pointed at
      independent physical Redis instances?
- [ ] Does a remote app being routed to via Gateway actually serve routes
      prefixed with its own `name` — or does it need `defaultRemoteApp`
      instead, as the one whole-domain default for this Gateway?
- [ ] Is `localAppNames` kept in sync with the real `defs` list passed to
      `activateApps`, so a locally-mounted app is never accidentally routed
      remotely?

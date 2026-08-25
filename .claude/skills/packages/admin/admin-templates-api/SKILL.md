---
name: admin-templates-api
description: The templates API's two-controller composition (CRUD owned by @zanix/notifications, POST /templates/sync owned by this package), the sync endpoint's two-Discovery-resource preference order and planCodeSync reconciliation, its concurrency-safe upsert semantics, and createTemplatesDiscoveryGuard shared across two Discovery endpoints. Use when working with /templates, the sync endpoint, or templates Discovery access.
---

Covers the Templates side of `zanix-admin`'s dual API surface. For the
Triggers side (a different composition, aggregator not central owner), see
`admin-triggers-aggregator`. File:line references point at
`~/Documents/Development/ZanixLibraries/admin` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- The CRUD half of `/templates` isn't this package's code at all — for
  behavior questions about `GET`/`POST`/`PUT`/`DELETE` themselves, the
  answer lives in `@zanix/notifications/templates-api`, not here. This
  skill's own territory is composition + the `sync` extension.
- Trust `planCodeSync`'s reconciliation rules rather than re-deriving
  seed/resync/leave-alone semantics per call site.

## Two controllers, one prefix

The templates API (`/templates`) is actually two separate controllers
composed under the same route prefix:

- **CRUD** (`GET`/`POST`/`PUT`/`DELETE`) — authored and owned end-to-end by
  `@zanix/notifications` (`@zanix/notifications/templates-api`'s
  `createTemplatesController` — schema/collection, RTOs, and the HTTP
  surface itself), the same "local API lives with its domain" shape
  `@zanix/datamaster` establishes for triggers. `zanix-admin` supplies the
  auth guard/protocol config at composition time — the controller itself
  never assumes an auth mechanism.
- **`sync`** (`POST /templates/sync`) — genuinely authored by this package
  (`createTemplatesSyncController`), since it needs
  `ServiceRegistry`/cross-service Discovery, a concept `@zanix/notifications`
  deliberately doesn't know about.

`@zanix/core`'s own built-in `/admin/templates` composes the exact same
pair, so the wire shape is identical either way. Requires `ADMIN_ROLE`/
`ADMIN_TEMPLATES_ROLE`, accepts either a human's `type: 'user'` token or a
machine's `type: 'api'` one. `ZanixAdminHub.start()`'s own `/templates`
(the hub route this skill covers) is bound to the `'admin-hub'` Application
by default, anchored whenever `ADMIN_HUB_SERVER_ID` is set — there's no
auto-generated anchored id. (The business-service-side embedded
`/admin/templates` — `defineAdminMetadata`, a different route — binds to
`'admin'`/`ADMIN_SERVER_ID` instead; don't conflate the two.)

## `POST /templates/sync`: batch, upsert-aware code pull

```ts
// Body: { serviceId: string }
// Response: { seeded: number; resynced: number }
```

This package's own `syncTemplatesFromRegisteredService` — cross-service
orchestration, not part of `TemplatesAdminService`, since it depends on
`ServiceRegistry`/Discovery-client concepts this package owns, not
`@zanix/notifications`. Typically triggered by a caller with **no local
database access of its own** (e.g. `@zanix/notifications`'s
`RemoteTemplateBackend`, Mode C) — instead of pushing templates as a
payload, it tells this endpoint *which registered service* to pull from.

`serviceId` is looked up in the shared `ServiceRegistry`
(`admin-service-registry`), resolving that service's base URL. This
package then pulls from whichever of two Discovery resources that service
exposes, **preferring the richer one**:

1. **`/.well-known/zanix/templates`** — that service's own DB-backed
   Discovery (this package's own, only present when the target has
   `admin` + DB-backed templates enabled) — real, currently-live content,
   including any manual edit. Tried first.
2. **`/.well-known/zanix/code-templates`** — `@zanix/notifications`'s own
   Discovery (the static in-code catalog only). Used whenever resource 1
   specifically isn't reachable (not registered, no DB-backed templates,
   or this service's own credentials aren't authorized for it) — present
   on any service using `@zanix/notifications`, regardless of DB-backed
   templates.

Either way, fetched entries reconcile against this service's own database
using the same `planCodeSync` (`@zanix/helpers`) rules
`LocalTemplateBackend` applies locally: seed a brand-new `{channel,name}`,
resync one nobody's edited directly since the last sync, leave a
manually-edited one alone, flip an entry no longer in the fetched set to
`source: 'database'` (never delete it). The merge doesn't care which of
the two resources the entries came from — a target's real DB-backed
content is simply treated as this service's own authoritative default,
same as its code catalog always has been.

**Additive, never a replacement** — `create()`/`update()` keep their
existing throw-on-conflict, human-facing CRUD semantics unchanged.

**Concurrency-safe by construction**: safe to call concurrently from N
replicas of the same service — each seed is a single atomic
`updateOne(..., {upsert: true})`, so two replicas racing the same
brand-new `{channel,name}` either both settle onto the same inserted row
(only the one that actually inserted is counted in `seeded`) or one hits
the collection's unique `{channel,name}` index as a duplicate-key error,
caught and treated as "already seeded," never surfaced as a failure.

`TemplatesAdminClient.sync(serviceId)` POSTs to this route for
`@zanix/admin`'s own internal callers — but `RemoteTemplateBackend` does
**not** use it: it hand-rolls its own POST instead, since importing
`TemplatesAdminClient` from `@zanix/notifications` would be circular
(`@zanix/admin` already depends on `@zanix/notifications` for
`ZanixTemplateAttrs`/`Notifiers`). Either way, the central service must
have `RemoteTemplateBackend`'s own `serviceId` registered in its
`ServiceRegistry`.

## `createTemplatesDiscoveryGuard`: one guard, two Discovery endpoints

```ts
import { createTemplatesDiscoveryGuard } from 'jsr:@zanix/admin@[version]'
const guard = createTemplatesDiscoveryGuard()
```

Builds the default `ADMIN_ROLE`/`ADMIN_TEMPLATES_ROLE` guard this
package's own `/.well-known/zanix/templates` Discovery endpoint requires —
exported so `@zanix/core`'s own `codeTemplatesDiscovery` option (the
static, in-code catalog at `/.well-known/zanix/code-templates`) can
require the exact same role, instead of re-inlining an equivalent
`jwtValidationGuard(...)` call that could quietly drift out of sync over
time. The two Discovery resources are different data (live DB-backed
records vs. a static in-code catalog), but "who's allowed to read a
template list" is the same question either way.

## Checklist before touching the templates API or sync endpoint

- [ ] Is a CRUD-behavior question actually being answered from
      `@zanix/notifications/templates-api`'s own docs, not assumed to live
      in this package?
- [ ] Does a new sync source respect the two-resource preference order
      (live DB-backed Discovery first, static code catalog fallback), not
      just whichever is convenient to fetch?
- [ ] Is the target service's `serviceId` actually registered in the
      shared `ServiceRegistry` before assuming `sync` can reach it?
- [ ] If adding a new Discovery-guarded endpoint tied to templates, does it
      reuse `createTemplatesDiscoveryGuard()` rather than a re-inlined
      equivalent guard?

---
name: core-admin-apis
description: Zanix.start({admin}) — the four built-in admin routes, port/prefix behavior on single-port platforms, Application rebinding (ADMIN_TRIGGERS_APPLICATION/ADMIN_TEMPLATES_APPLICATION/ADMIN_DLQ_APPLICATION), roles/protocol header, ADMIN_SERVER_ID pinning + rotation, codeTemplatesDiscovery, and the re-exported pieces for building a custom admin API. Use when enabling/configuring this service's own embedded admin server.
---

Covers `@zanix/core`'s built-in, per-service admin server — distinct from
the centralized `ZanixAdminHub` orchestrator, see `core-admin-architecture`
for that relationship. File:line references point at
`~/Documents/Development/ZanixLibraries/core` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- `admin` is disabled by default — check whether a task actually needs
  this service's own embedded admin API before assuming it's already
  there.
- `ADMIN_SERVER_ID` pinning is the safer default over rebinding a route to
  `'main'` — check the "Application rebinding" section before reaching for
  the latter.

## Enabling it

```ts
await Zanix.start() // no admin server (default)
await Zanix.start({ admin: true }) // enabled, shares server's own per-type port by default
await Zanix.start({ admin: { rest: { port: 4000 } } }) // enabled, explicit REST port
```

Bootstraps a second, **anchored, `'admin'`-Application** server, never
reachable through the public `server` options. `admin` (`false`/`undefined`
by default) accepts the same per-type shape `server` does
(`rest`/`graphql`/`socket`/`ssr`), **except `application`** — always bound
to `'admin'`, so passing it is a type error, not a silently overridden
value. `@zanix/admin` composes no `ssr` routes of its own, so `admin.ssr`
only matters if your own app composes an `ssr` handler under the shared
`'admin'` Application too. `admin.health` is a sibling of the per-type
fields (like top-level `health`) — explicit here always wins; omitted, it
inherits `server.health`.

## The four built-in routes

- **`/admin/triggers`** — manages `@zanix/datamaster`'s persisted trigger
  configs. Always registered once `admin` is enabled (`TRIGGERS_MODEL_NAME=false`
  disables). Underlying service/repository owned by `@zanix/datamaster`.
- **`/admin/templates`** — manages `@zanix/notifications`'s Handlebars
  templates. Only registered once `templatesBackendMode() === 'local'` (i.e.
  `TEMPLATES_BACKEND=local` is selected) — `TEMPLATES_MODEL_NAME` alone, with
  the selector unset, does NOT enable this. Owned by `@zanix/notifications`.
- **`/admin/dlq`** — manages `@zanix/datamaster`'s persisted Dead Letter
  Queue entries. Opt-in, registered only when `DLQ_MODEL_NAME` is set —
  deliberately NOT triggers' on-by-default-unless-disabled shape, since
  `registerDLQModel()` is a standalone call a host's own bootstrap must make
  explicitly, unlike the triggers model. Owned by `@zanix/datamaster`.
- **`/admin/service-token`** — machine-to-machine credential exchange
  (`exchangeServiceCredential`). Always registered whenever `admin` is
  enabled, **unauthenticated by design** — there's no session to gate yet,
  the caller is here to obtain one.

Triggers and templates are re-exported from `@zanix/core`'s own `mod.ts`
unchanged (`@zanix/admin` is the real owner) — a consuming app can reuse the
same classes to build its own custom triggers/templates API instead of
duplicating the CRUD logic. DLQ's own admin pieces (`createDlqAdminController`
and friends) are not re-exported from `@zanix/core`'s `mod.ts` yet — only the
route itself is wired in via `defineLocalAdminApp()`; see "Building a custom
admin API instead" below. Read-only `/.well-known/zanix/templates`/
`/.well-known/zanix/triggers`/`/.well-known/zanix/dlq` Discovery endpoints
are also registered on the same admin server, gated by the same role pair as
their CRUD counterpart, for a central sync job or aggregator to snapshot
this service's resources without the authenticated CRUD surface.

## Ports and single-port platforms

Each admin sub-server defaults to **the same port `server`'s own config
resolves to for that type** — sharing one real listener with the main
server by default, dispatched by URL path. Without `ADMIN_SERVER_ID`, this
bootstrap gives the admin server its own distinct default `globalPrefix`
(`` `admin-${type}` ``) so the two never collide on a shared port even
without opting into anchoring.

Two ways to get a genuinely separate port:
- **`PORT_<TYPE>`/`PORT`** — an env var, if set, wins over everything for
  that type, applied uniformly to **both** the main and admin server (both
  are still `type: 'rest'`). Couples the two ports together — use an
  explicit `admin.<type>.port` instead if they need to differ.
- **`admin.<type>.port`** — an explicit code value always wins over the
  "reuse `server`'s port" default (though `PORT`/`PORT_<TYPE>` still wins
  over this too).

No separate `ADMIN_PORT`-style env var exists or is planned — it would be a
second, competing axis against `PORT`/`PORT_<TYPE>` with no clear
precedence story.

## Application rebinding: `ADMIN_TRIGGERS_APPLICATION`/`ADMIN_TEMPLATES_APPLICATION`/`ADMIN_DLQ_APPLICATION`

```env
ADMIN_TRIGGERS_APPLICATION=main
ADMIN_TEMPLATES_APPLICATION=main
ADMIN_DLQ_APPLICATION=main
```

Moves that one capability onto a different Application's own server
instead of `'admin'` — e.g. `'main'` moves it onto the same
`bootstrapServers(options.server)` call your own app's routes serve from,
reachable at its normal prefix instead of the admin server's own. Leaving
all three unset (default) keeps all three capabilities on `'admin'`.

**This is a Runtime-binding override, never an authentication/authorization
one** — rebinding `/admin/triggers` onto `'main'` only changes *where* it's
served; `AuthTokenValidation` and the role gate below remain the actual
access-control boundary either way. Use this only if the deployment
platform genuinely can't isolate the admin server on its own address —
pinning `ADMIN_SERVER_ID` on `'admin'` (the default) is the safer,
recommended choice otherwise: an unguessable, operator-chosen URL prefix as
defense-in-depth underneath the real auth gate.

**Caveat: `'admin'` isn't exclusively reserved for this package.**
`@zanix/server` keeps exactly one route bucket per Application name per
server type — every capability composed under `'admin'`, regardless of
which package composed it, shares that bucket. If your own app also
composes routes under `'admin'` in the same process as `admin: true`, both
sets get mounted together — and potentially served twice, once under each
call's own id/prefix, if you also bootstrap it a second time yourself.

## Roles, authentication, and the protocol header

```ts
import {
  ADMIN_DLQ_ROLE,
  ADMIN_ROLE,
  ADMIN_TEMPLATES_ROLE,
  ADMIN_TRIGGERS_ROLE,
} from 'jsr:@zanix/core@[version]'

await authProvider.session.generateTokens(adminUser, { permissions: [ADMIN_ROLE] }) // grants all three
```

`ADMIN_ROLE` grants all three APIs; `ADMIN_TRIGGERS_ROLE`/
`ADMIN_TEMPLATES_ROLE`/`ADMIN_DLQ_ROLE` grant just one each — permissions
are OR'd, either role is enough on its own. Every route accepts a human
admin's `type: 'user'` token (`Authorization: Bearer <token>`) or a machine
caller's `type: 'api'` one (`X-Znx-Authorization: Bearer <token>`) on the
same route.

Every response from all four APIs carries `X-Znx-Admin-Protocol`
(`ADMIN_PROTOCOL_HEADER`, currently version `1`) — validates a caller's own
declared version against what it still understands, rejecting an
unrecognized one with `400 Bad Request` rather than silently guessing. No
caller sends this header today, so an absent declared version always
defaults to current — nothing changes for an existing consumer.

## Pinning a stable address: `ADMIN_SERVER_ID`

```env
ADMIN_SERVER_ID=custom-billing
```

Without it, the admin server is a plain, unprefixed server — no
auto-generated/random anchored id. Setting it gives an external caller
(e.g. `@zanix/admin`'s `ServiceRegistry`) a stable address to reach this
service at. **There is no discovery mechanism for the id, by design, and
none is planned** — reachability is always opt-in via pinning; the only job
an unset `ADMIN_SERVER_ID` serves is raising the cost for a
credential-less network-adjacent attacker, not waiting for some future
caller to learn it.

**Rotating a pinned id — `ADMIN_SERVER_ID_PREVIOUS`**:

```env
ADMIN_SERVER_ID=billing-v2
ADMIN_SERVER_ID_PREVIOUS=billing-v1
```

Both prefixes reach the same routes simultaneously for as long as
`ADMIN_SERVER_ID_PREVIOUS` stays set — update callers' own config at your
own pace, then drop the env var in a later redeploy. **Not supported for
the admin GraphQL sub-server** — rebuilding its schema for a second prefix
would compile an empty stub instead of the real one; REST/socket rotate
independently of that limitation.

## `codeTemplatesDiscovery`

```ts
Zanix.start({ codeTemplatesDiscovery: true }) // or an object to override guards/application
```

Separate from everything above — exposes this service's own in-code
templates catalog (`@zanix/notifications`'s `CODE_TEMPLATES`) at
`/.well-known/zanix/code-templates`, so an external central service can
pull them as seed data. **Disabled by default, deliberately independent of
`TEMPLATES_SERVICE_URL`/Mode C** — consuming templates from a remote source
doesn't imply agreeing to expose your own catalog back. `true` guards it
with the same role that already protects `/.well-known/zanix/templates`;
pass `guards: []` to deliberately serve it unauthenticated — never omit
`guards` silently to get that effect.

## Building a custom admin API instead

`@zanix/core` re-exports the lower-level pieces the built-in routes are
built from, so a custom controller reuses the same CRUD logic:
`createTriggersAdminController`, `TriggersAdminService`/
`TriggersAdminRepository`, `TemplatesAdminService`/
`TemplatesAdminRepository`, the CRUD RTOs, `ServiceExchangeRTO`,
`ADMIN_PROTOCOL_SUPPORTED_VERSIONS`.

## Checklist before enabling/configuring the admin server

- [ ] Is `ADMIN_SERVER_ID` set only when something external genuinely needs
      a fixed address — not left set out of habit?
- [ ] Is `ADMIN_TRIGGERS_APPLICATION`/`ADMIN_TEMPLATES_APPLICATION`/
      `ADMIN_DLQ_APPLICATION` rebinding used only when the deployment
      platform truly can't isolate the admin server — not as a default
      choice over pinning?
- [ ] If this app also composes its own routes under the `'admin'`
      Application, is that deliberate, given they'll share the same bucket
      as `@zanix/admin`'s own routes?
- [ ] Is `codeTemplatesDiscovery` enabled only once this service actually
      wants an external service pulling its catalog?

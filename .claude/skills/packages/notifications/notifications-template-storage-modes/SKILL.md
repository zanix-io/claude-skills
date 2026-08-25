---
name: notifications-template-storage-modes
description: Where @zanix/notifications resolves template content from — pure code (default), a local @zanix/datamaster-backed database (per-service or shared across services), or a fully remote central Template Service (Mode C) — plus the local /templates CRUD API. Use when enabling database-backed templates, setting up Mode C for a service with no local Mongo access, or reviewing the templates-api local controller.
---

Builds on `notifications-templates`'s base rendering system and
`notifications-template-inheritance`'s parent-chain mechanism — this skill
covers *where the content resolves from*, not the rendering/inheritance
mechanics themselves. File:line references point at
`~/Documents/Development/ZanixLibraries/notifications` — read the real code
there before assuming this summary is still accurate.

## Golden rule (token savings)

- **`TEMPLATES_BACKEND` is the single selector deciding the mode** —
  `'local'` (Modes A/B below), `'remote'` (Mode C), or unset (pure code).
  Identify which value it's set to before reading anything else in this
  skill — each mode's section is self-contained; you rarely need more than
  one for a given task. A mode's own vars (`TEMPLATES_MODEL_NAME` for
  local; `TEMPLATES_SERVICE_URL`/`_ID`/etc. for remote) are only ever
  consulted once `TEMPLATES_BACKEND` actually selects that mode — setting
  one mode's var while a different mode (or none) is selected simply has no
  effect, it's never read.
- For Mode C's vars, validate them together with the one `curl`
  smoke-test command below rather than tracing through `RemoteTemplateBackend`'s
  source to predict whether they're configured correctly.

## Default: pure code, no database access at all

Every template renders from compiled code unless `TEMPLATES_BACKEND` is set
to `'local'` or `'remote'`. Nothing here to configure.

## Mode A — per-service database-backed (the common case)

Setting `TEMPLATES_BACKEND=local` switches `TemplateProvider` to a
hybrid mode (`TEMPLATES_MODEL_NAME` is optional under this mode, defaulting
to `zanix-templates`): on first use, every code template seeds into a `ZanixTemplate`
collection (`{channel, name}` unique index, `source: 'code'`); from then on,
a `{channel, name}` with a database record renders from that record's live
`hbs` instead of the compiled version — an edit through an admin CRUD API
takes effect on the very next send, no redeploy.

- **A later code change only re-syncs if nobody has edited the database
  record since the last sync** — a manual database edit always wins over a
  subsequent code change, with no exception.
- **A template removed from code is never deleted from the database** — it
  flips to `source: 'database'` and keeps rendering exactly as last synced.
- **Any failure on the database path falls back to the code version with a
  logged warning** — enabling this is meant to be strictly additive, never a
  new way for a send to fail.
- **Clearing `TEMPLATES_BACKEND` always wins, no separate kill switch
  needed** — falls back to pure code even with a real matching database
  record and `TEMPLATES_MODEL_NAME` still set, since the selector alone
  decides. (There used to be a separate `DATABASE_TEMPLATES=false` kill
  switch, pre-selector — removed entirely, not just superseded; don't cite
  it as current.)

Requires a real `@zanix/datamaster` `ZanixMongoConnector`, registered under
the `'database'` core-provider key — `TemplateProvider` is typed directly
against it; there's no other backend to be storage-agnostic about.

### `name` vs `hash` — easy to conflate, entirely different purposes

- **`name`** identifies the template within its `channel` — the compound
  unique key is `{channel, name}`, and it's exactly the string passed as
  `zanixTemplate`. No automatic derivation; it must match on both the
  persisted record and every call site.
- **`hash`** is purely a cache-invalidation key for `TemplateProvider`'s
  in-memory compiled-render cache — **unrelated to the code↔database sync
  decision**, which compares real `hbs` content directly, never `hash`. It
  can be any string — nothing validates it as a real hash — the only
  requirement is that it *changes* whenever `hbs` changes; otherwise the
  cache keeps serving the stale compiled version, silently.
  - For code-synced records, a real SHA-256 of `hbs` is computed
    automatically — never set this yourself.
  - For a manual edit or a new database-only template, **you must set
    `hash` yourself alongside `hbs`** — `@zanix/utils`'s `generateUUID()` is
    a convenient, guaranteed-distinct choice.

```ts
await Model.updateOne(
  { channel: 'email', name: 'generic' },
  { $set: { hbs, hash: generateUUID() } }, // hash MUST change or the edit won't be picked up
)
```

### Multi-instance behavior

The sync memo and compiled-render cache are per-process, not shared across
replicas — each instance syncs/compiles independently, but a manual edit's
`hbs`/`hash` is read fresh from the database on every `resolve()` call
(never cached across calls), so it's picked up on the next request in every
instance, no coordination needed. The one edge case: two replicas seeding
the same new template at nearly the same moment — the second hits the
`{channel, name}` unique-index conflict, falls back to code once for that
call, and self-heals on the next.

## Mode B — shared across services

`TemplateProvider` always reuses the app's own `'database'` core connector —
no dedicated connection for templates. `registerModel()`'s `'db:name'`
multi-database format (see `datamaster-database-and-models`) can target a
**different database on the same Mongo cluster**: setting
`TEMPLATES_MODEL_NAME` to the same `'sharedDb:zanix-templates'` value across
every participating service makes them all read/write the same collection,
regardless of each service's own default database — no new code required.

**This is a deliberate, conscious exception, not a default path** —
`@zanix/datamaster`'s own docs flag multi-database naming as "not recommended
for microservices," and that guidance still holds generally (see
`datamaster-database-and-models`); Mode B only makes sense when every
participating service genuinely shares the same Mongo cluster and the
templates genuinely need to be shared, not as a convenience default.

## Mode C — remote-only, no local database access at all

For a service with **zero local Mongo access to templates**:

```ts
Deno.env.set('TEMPLATES_BACKEND', 'remote')
Deno.env.set('TEMPLATES_SERVICE_URL', 'https://templates.internal.example')
Deno.env.set('TEMPLATES_SERVICE_ID', 'billing')
Deno.env.set('TEMPLATES_SERVICE_TOKEN', myPreIssuedApiToken)
```

**`TEMPLATES_BACKEND=remote` is what actually selects this mode — the URL/ID/
token alone are never enough.** Setting `TEMPLATES_SERVICE_URL`/`_ID` without
`TEMPLATES_BACKEND=remote` has no effect at all (confirmed: not a throw, a
genuine no-op — the selector, not the URL, decides). Conversely,
`TEMPLATES_MODEL_NAME` set alongside `TEMPLATES_BACKEND=remote` is also
simply never consulted — Mode A's vars only matter under `'local'`. Matters
in practice when two processes read the same `.env` file with opposite
needs (a Mode C consumer and the central service itself, which needs
`TEMPLATES_BACKEND=local`) — set `TEMPLATES_BACKEND` explicitly per
process's own entrypoint rather than relying on one shared file.

- **`TEMPLATES_SERVICE_URL`** — the central service's *admin* base URL. No
  `/admin/templates` suffix; `TemplateProvider` appends it per call.
- **`TEMPLATES_SERVICE_ID`** — this service's own identity in the central
  service's `ServiceRegistry`. Required alongside `TEMPLATES_SERVICE_URL`.
- **`TEMPLATES_SERVICE_TOKEN`** — a pre-issued machine credential
  (`X-Znx-Authorization: Bearer <token>`, `@zanix/auth`'s `type: 'api'`
  contract, RS256). This package never mints it; issuing/rotating it is the
  operator's/central service's job. **Takes priority when set** — the
  dynamic exchange below is never attempted if this is present. The only
  option that works against a central service outside the Zanix ecosystem.
- **`TEMPLATES_SERVICE_AUTH_ID`** — the dynamic alternative, for a central
  service that IS itself Zanix-based: signs a short-lived assertion and
  exchanges it automatically via `@zanix/auth`'s `createServiceAuthClient`
  (same primitive `ZanixAdminHub.start({ auth })` uses). No static token to
  rotate by hand. **This is the service's own signing identity (`iss`/`sub`)
  — distinct from `TEMPLATES_SERVICE_ID`**, a routing key the central
  service's registry uses; the two are independent even if nothing stops you
  choosing the same string for both. Keys resolve automatically via
  `@zanix/auth`'s own convention: `JWK_PRI_<TEMPLATES_SERVICE_AUTH_ID>[_<keyId>]`
  for signing, `JWK_ID_<TEMPLATES_SERVICE_AUTH_ID>` for which key to use on
  rotation — the mirror image of the *verifying* side's
  `JWK_PUB_<serviceId>[_<keyId>]` convention. Base64-encoded, PKCS#8 only —
  `generateRSAKeys()` from `@zanix/helpers`, store `btoa(privateKey)`, never
  the raw PEM. To rotate: register the new keypair under a new `_<newId>`
  suffix alongside the current one, then flip `JWK_ID_<TEMPLATES_SERVICE_AUTH_ID>`
  to `<newId>` — a config change, with a real overlap window, not a code
  change.
- **`TEMPLATES_SERVICE_CACHE_TTL_MS`** — overrides the default 45-second
  local cache TTL on the remote fetch (separate from the compiled-render
  cache) — a remote outage/latency spike doesn't turn every `resolve()` into
  a blocking round-trip; a stale-but-recent copy is an acceptable
  availability trade-off, the same reasoning as a DNS/config cache.

**Validate the three vars together before traffic** — `RemoteTemplateBackend`'s
own self-check catches and logs failures without throwing, but only fires
lazily on this process's *first* `resolve()` call. For a deploy-pipeline
smoke test:

```sh
curl -X POST "$TEMPLATES_SERVICE_URL/admin/templates/sync" \
  -H "X-Znx-Authorization: Bearer $TEMPLATES_SERVICE_TOKEN" \
  -H 'Content-Type: application/json' \
  -d "{\"serviceId\":\"$TEMPLATES_SERVICE_ID\"}"
# expect a 2xx — validates all three together, with zero new code
```

**Remote sync is pull-based, best-effort, at most once per process.** On the
first `resolve()` call, `RemoteTemplateBackend` fires a single `POST
admin/templates/sync` naming which service to pull from — template content
itself is never sent as a request body. This requires exposing this
process's own `CODE_TEMPLATES` via `defineCodeTemplatesDiscovery()`:

```ts
await ProgramModule.defineApplication('main', () => {
  defineCodeTemplatesDiscovery() // serves /.well-known/zanix/code-templates
})
```

Accepts an optional `{ guards }` — this package has no dependency on
`@zanix/auth`, so it never assumes an auth scheme; pass a guard from your own
bootstrap if this endpoint should require one. Any failure (network error,
non-2xx, unregistered `serviceId`, an older central service that doesn't
expose the route) is caught and logged as a warning, **never rethrown, and
never retried within the same process** — seeding the central database is an
enhancement on the read path, never a new hard dependency.

**Composes automatically with `RestClient`'s conditional-`GET` support** — on
top of the local TTL cache, every remote fetch already goes through
`ETag`/`If-None-Match` handling once the central service returns an `ETag`
header (naturally `ZanixTemplateAttrs.hash`) — no code change needed in this
package for that.

## The local `/templates` CRUD API

This package owns the full local `/templates` CRUD API — data access,
business logic (`TemplatesAdminRepository`/`TemplatesAdminService`), and the
HTTP surface fronting them (`@zanix/notifications/templates-api`'s
`createTemplatesController`) — the same "local API lives with its domain"
shape `datamaster-triggers`/`zanix-local-api-vs-aggregator` establish. See
`zanix-local-api-implementation` for the mechanics this controller follows.
`@zanix/admin` separately composes a genuinely cross-service extension on
top (`POST /templates/sync`, pulling a registered service's own code
templates via `ServiceRegistry`/Discovery) — owned and authored by
`@zanix/admin` itself, mounted under the same `/templates` prefix, since
that's the part needing cross-service concerns this package deliberately
knows nothing about.

```ts
import { TemplatesAdminRepository } from 'jsr:@zanix/notifications'
import { createTemplatesController } from 'jsr:@zanix/notifications/templates-api'
import { jwtValidationGuard } from 'jsr:@zanix/auth'

class MyCustomTemplatesRepository extends TemplatesAdminRepository {
  // extend/override — base CRUD (list/get/create/update/remove) is already
  // correct with respect to source/version/hash/soft-delete semantics.
}

createTemplatesController({
  guards: [jwtValidationGuard({ permissions: ['my-app:templates'], type: ['user'] })],
})
```

`createTemplatesController` never assumes an auth mechanism — pass real
guards explicitly, per `zanix-server-internals`'s "auth is never assumed"
rule.

## Checklist before configuring template storage

- [ ] Which mode does the deployment actually need — pure code, Mode A (this
      service's own data), Mode B (a genuine multi-service shared catalog on
      the same cluster), or Mode C (no local Mongo at all)? Don't default to
      Mode A just because it's the common case if the real requirement is
      Mode C.
- [ ] Is `hash` set (and changed) alongside every manual `hbs` edit — an edit
      without a `hash` change is silently ignored by the render cache?
- [ ] For Mode C, were the three required env vars validated together via
      the smoke-test `curl`, not assumed correct from individually setting
      them?
- [ ] Does `defineCodeTemplatesDiscovery()` actually get called inside an
      Application scope, if Mode C's remote-sync-from-code-templates
      capability is expected to work?
- [ ] Does a new/custom `/templates` controller pass real `guards` explicitly
      — never assuming `createTemplatesController` provides auth by default?

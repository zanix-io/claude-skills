---
name: zanix-local-api-vs-aggregator
description: Where a new HTTP controller for a capability's own data belongs — inside the owning domain package as a "local API," or in @zanix/admin as a cross-service aggregator. Use before writing a new controller in any Zanix library package, or before deciding whether @zanix/admin should own a new resource's CRUD surface.
---

This rule governs one specific, recurring design question: when a Zanix library
package's own data needs an HTTP surface, does that surface live with the data,
or does it live in `@zanix/admin`? It's a direct consequence of the dependency
tiers in `zanix-dependency-direction` (an application-tier package like
`@zanix/admin` may depend downward on any library's concrete types, but that
doesn't make it the rightful home for every HTTP surface those libraries need)
— split into its own skill because `@zanix/datamaster`, `@zanix/notifications`,
and `@zanix/space` each cite this exact rule by name in their own `docs/`.
File:line references point at the actual precedent in this monorepo — read the
real code before assuming this summary is still accurate; it will drift as the
packages evolve.

## Golden rule (token savings)

- The decision criterion (local-API vs. aggregator) is a one-question check —
  "does it need `ServiceRegistry`/Discovery/cross-service fan-out?" — apply it
  directly instead of walking the reference implementations again for a
  controller whose answer is already obvious from that question alone.
- Only pull up the `assets-api`/`triggers-api`/`templates-api` reference
  implementations when a real ambiguous case needs one to compare against —
  not as a mandatory step for every review.

## Ownership discipline: composing ≠ owning

A package that only *wires* a capability into a bootstrap sequence is not that
capability's owner. The owner is whoever holds the actual data/schema at
runtime. Concretely:

- `@zanix/admin` composing `createTriggersAdminController()`/
  `createTemplatesDiscoveryProvider()` inside its own `defineAdminMetadata()`
  does not make `@zanix/admin` the templates/triggers domain owner — it's the
  composer. `@zanix/notifications` (templates schema) and `@zanix/datamaster`
  (triggers persistence) are the real data owners.
- `TemplatesAdminRepository`/`TemplatesAdminService` live in
  `@zanix/notifications`
  (`notifications/src/modules/templates/db/{templates.repository,templates.service}.ts`);
  `TriggersAdminRepository`/`TriggersAdminService` live in `@zanix/datamaster`
  (`datamaster/src/modules/triggers/{triggers.repository,triggers.service}.ts`).
  `@zanix/admin` composes both but does not author either. Both packages' own
  docs (`notifications/docs/templates.md`, `datamaster/docs/triggers.md`)
  describe this as the current, intentional arrangement.
- When adding a NEW resource kind to any cross-cutting mechanism (Discovery, or
  its future successors), put the actual query/business logic in the package
  that owns the underlying data, and have the composing package (`@zanix/admin`,
  `@zanix/core`) only import and register it. Don't default to writing it in the
  composer just because that's where an existing CRUD layer happens to live —
  that repeats the gap this rule exists to close.

## The rule

```
domain/application/persistence   →  never imports @zanix/server's Controller family
                                     (Controller, ZanixController, Get/Post/Put/Delete, HandlerContext)
local-api (same package)         →  may import @zanix/server; CRUD 1:1 over data the package already owns
aggregator/hub (@zanix/admin)    →  composes local-apis; owns ServiceRegistry/Discovery/cross-service fan-out
```

**Decision criterion for where a new HTTP controller belongs** — ask this before
writing it, not after:

- Does it do CRUD 1:1 over data the domain package already owns and persists,
  with no cross-service concern? → it's a **local API**. It belongs inside that
  same package, next to the domain/service/repository it fronts (e.g.
  `templates-api/`, `triggers-api/`), and that subpath — and only that subpath —
  is allowed to import `@zanix/server`'s Controller family.
- Does it need `ServiceRegistry`, discover or iterate over multiple services, or
  proxy HTTP calls to another process's own local API? → it's an **aggregator**.
  It belongs in `@zanix/admin`, and nowhere else.
- **"It has historically lived in `@zanix/admin`" is never on its own a reason
  to put a new local API there, or to leave an existing one there.** Admin being
  the composer of a resource does not make admin that resource's rightful home
  for its CRUD surface.

## Reference implementations — already fully conformant

- **`@zanix/space`'s `assets-api`**:
  `assets-api/controllers/assets.controller.ts` imports `@zanix/server`
  (`Controller`, `Get`, `Post`, `Guard`, `ZanixController`, `HandlerContext`).
  `asset-transform/` and `media/` (the domain/transform layers `assets-api`
  fronts) import `@zanix/server` **nowhere** — verified empirically, not
  assumed. `assets-api` depends on `asset-transform`; `asset-transform` never
  depends back — enforced automatically, not just by convention, via
  `space/src/@tests/unit/asset-transform/dependency-boundary.test.ts` and its
  `assets-api`-side mirror, both of which walk the real `deno info --json`
  module graph (BFS) to prove the boundary holds. **This test pattern is the
  reusable template** for enforcing the same boundary anywhere else this rule
  applies — don't reinvent a different mechanism; see
  `zanix-local-api-implementation` for the technique itself, copy-adaptable,
  once you're actually building one. `space/deno.jsonc` exposes
  `./assets`, `./assets-api`, and `./media` as separate public subpaths, with
  the layering rule spelled out directly in the `"exports"` map's own inline
  comment.
- **`@zanix/datamaster`'s `triggers-api`**: `createTriggersAdminController`
  (`@zanix/datamaster/triggers-api`) is a real, authenticatable `/admin/triggers`
  HTTP API over `TriggersAdminService`, built the same way `assets-api` fronts
  its own domain layer — this package owns both the data and the local HTTP
  surface fronting it, never assuming an auth mechanism itself (per
  `datamaster/docs/triggers.md`).
- **`@zanix/notifications`'s `templates-api`**: `createTemplatesController`
  (`@zanix/notifications/templates-api`) owns the full local `/templates` CRUD
  API — data access, business logic
  (`TemplatesAdminRepository`/`TemplatesAdminService`), and the HTTP surface
  fronting them — the same shape as the other two (per
  `notifications/docs/templates.md`, which cites this exact rule by name).

## What correctly stays in `@zanix/admin` under this rule, and why

`TriggersAggregator` (`admin/src/modules/triggers/triggers.aggregator.ts`) and
the hub `triggers.handler.ts` are genuine aggregators — `TriggersAggregator`
never touches a local database at all; it fans out over a `ServiceRegistry` and
proxies HTTP calls to each remote service's own `/admin/triggers` local API and
its `/.well-known/zanix/triggers` Discovery snapshot. That is exactly the
cross-service shape this rule reserves for `@zanix/admin`. The same holds for
`@zanix/admin`'s `POST /templates/sync` (`syncTemplatesFromRegisteredService`):
it pulls a registered service's own code templates via `ServiceRegistry`/
Discovery, a genuinely cross-service concern `@zanix/notifications` deliberately
knows nothing about.

## A resource can have both a local-api half and an aggregator half at once

That's not a violation — it's this rule applied per-route rather than
per-resource. Templates' CRUD (`list`/`get`/`create`/`update`/`remove`) is pure
local-api shape; its `sync` route (`syncTemplatesFromRegisteredService`) depends
on `ServiceRegistry` and is genuinely aggregator-shaped, mounted under the same
route prefix but owned and authored by `@zanix/admin`. A resource whose routes
mix both shapes gets split along that line — the CRUD half lives in the domain
package, the cross-service half stays in `@zanix/admin` as its own small
controller — rather than leaving the whole thing in `@zanix/admin` because one
route out of five needs to stay there, or trying to force the cross-service
route into the domain
package to keep one controller class intact.

## Checklist before writing a new HTTP controller in a library package

- [ ] Does the domain/persistence layer this controller fronts import
      `@zanix/server`'s Controller family anywhere? It shouldn't — only the
      `-api` subpath should.
- [ ] Is there a dependency-boundary test (module-graph BFS, per the `assets-api`
      template — see `zanix-local-api-implementation` for the technique)
      proving the domain layer never depends back on the `-api` subpath?
- [ ] Does this controller need `ServiceRegistry`/Discovery/cross-service fan-out?
      If yes, it belongs in `@zanix/admin`, full stop — not "for now," not "since
      it's small."
- [ ] If a resource is being split (local CRUD vs. cross-service route), does
      each half live in the package the rule assigns it to, rather than both
      staying together in whichever package happened to have the code first?

## Out of scope — do not do these

- How to actually build a local API once you've decided a controller belongs
  in one — subpath layout, guard defaults, the dependency-boundary BFS test
  itself — that's `zanix-local-api-implementation`'s job; this skill only
  answers *where*, never *how*.
- The general dependency-tier rule this skill is one consequence of (what a
  tier is, when a direct downward import is valid vs. when inversion is
  required) — that's `zanix-dependency-direction`'s job; don't restate it
  here, cite it.
- Authenticating a local API's endpoints or an aggregator's proxied calls
  (guards, service-credential exchange) — that's
  `zanix-local-api-implementation`/`admin-service-authentication`'s job; this
  rule places the controller, it doesn't secure it.
- Deciding whether a new cross-cutting mechanism (Discovery, `ServiceRegistry`,
  a future successor) should exist at all — that's a design call made
  elsewhere; this skill only routes an already-decided HTTP surface to the
  right package once the mechanism exists.

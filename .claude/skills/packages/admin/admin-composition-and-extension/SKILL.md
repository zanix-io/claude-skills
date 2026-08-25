---
name: admin-composition-and-extension
description: ZanixAdminHub.start() vs. composing defineAdminHubApp()/defineLocalAdminApp() directly, and the Extension pattern this package is the ecosystem's own reference example of — adding a sub-app (its own operations/mcp surface, no REST routes of its own) instead of forking or editing the base app's manifest. Use when standing up a zanix-admin instance, or adding a new capability to admin/admin-hub without forking it.
---

Covers how `@zanix/admin` itself is composed, and the reference pattern for
extending a Zanix App without forking it. For the specific
Triggers/Templates surfaces this pattern was extracted from, see
`admin-triggers-aggregator`/`admin-templates-api`. File:line references
point at `~/Documents/Development/ZanixLibraries/admin` — read the real
code there before assuming this summary is still accurate.

## Golden rule (token savings)

- Use `ZanixAdminHub.start()` for the common case — reach for composing
  `defineAdminHubApp()`/`defineLocalAdminApp()` directly only when a host
  genuinely needs to install alongside its own other apps via `Zanix.start()`'s
  `apps` option or `activateApps` directly.
- Follow the 3-step sub-app template below verbatim for a new extension —
  don't invent a new composition primitive; this pattern deliberately reuses
  `activateApps()`'s existing "list of independent apps" contract.

## Three ways to run this package

**The quick path**: `ZanixAdminHub.start()` registers
`TriggersController`/`TemplatesController` plus their supporting
connectors/providers, starts a REST server, traps `SIGINT`/`SIGTERM`
automatically (no opt-out), and runs `ZanixAdminHub.stop()` before exiting
— a real standalone deployable service. Unlike `@zanix/core`'s
`Zanix.start()`, a `stop()` failure during this shutdown never force-exits
the process, since this package is often just one participant sharing a
process with an unrelated entrypoint.

**Manual wiring**: use `TriggersAggregator` directly for an app building
its own bootstrap instead of `ZanixAdminHub.start()`
(`admin-triggers-aggregator`).

**Composing the Zanix Apps directly**: `ZanixAdminHub.start()` and
`@zanix/core`'s own `admin: true` option are both thin wrappers over two
`@zanix/app` manifests this package exports — `defineAdminHubApp(options)`
(the central aggregator/proxy, `/triggers`/`/templates`) and
`defineLocalAdminApp()` (the embedded, business-service-side CRUD,
`/admin/triggers`/`/admin/templates`/`/admin/service-token`). A host
composing its own set of Zanix Apps can install either alongside its own
apps directly:

```ts
import Zanix from 'jsr:@zanix/core@[version]'
import { defineAdminHubApp } from 'jsr:@zanix/admin@[version]'

await Zanix.start({
  apps: {
    'admin-hub': {
      definition: defineAdminHubApp({ auth: { serviceId: 'zanix-admin-hub' } }),
      server: { rest: { port: 9000 } },
    },
  },
})
```

`defineAdminHubApp` declares one dependency, `registry` (type
`'service-registry'`) — override it via the host's own `resources`/`uses`
to share a single `ServiceRegistry` instance across this app and others in
the same process, instead of relying solely on `setServiceRegistry`
(`admin-service-registry`).

Both `defineAdminHubApp`/`defineLocalAdminApp` bind to the `'admin'`
Application and are anchored by default — this is `zanix-admin`'s own
admin/ops surface, not meant to be reachable by an arbitrary public caller.
Both accept a `prefix` override via the factory's own argument, plus an
`application: 'main'` override for a deployment platform that genuinely
can't isolate an anchored server.

## The Extension pattern — this package's reference example

`@zanix/admin` is the ecosystem's own reference example of the
**Extension** pattern for customizing a Zanix App without forking it:
adding capability that doesn't replace anything already there, as one or
more SEPARATE Zanix Apps composed alongside a base app, rather than a
change bolted onto the base app's own manifest. Contrast with **Override**
(replacing a piece of existing behavior via `resources`/`uses`,
`registerCoreProviderSlot`, or `@zanix/app`'s `behaviors`/`ctx.behavior()`)
— a different question this pattern doesn't answer (see `@zanix/app`'s own
`app-behaviors-and-overrides` for that side).

**The concrete, real illustration**: `defineAdminHubApp()`'s own
`/triggers`/`/templates`/`/dlq` REST surface owns no `operations`/`mcp`
invocation surface of its own — the `operations`/`mcp` surface other
apps/agents invoke via `ctx.remote('admin-hub').call(...)` lives entirely in
dedicated, physically-separate Zanix Apps instead — `defineHubTriggersApp()`/
`defineHubTemplatesApp()`/`defineHubDlqApp()` (hub side),
`defineLocalTriggersApp()`/`defineLocalTemplatesApp()`/`defineLocalDlqApp()`
(local side), each:

- Has its own `name` (its own Application/route-dispatch identity) and its
  own `routes: false` — a sub-app owns no REST surface of its own; the REST
  controller stays on the parent, only the `operations`/`mcp` invocation
  surface moved.
- Declares its own `operations`, ADDING a new invocation surface — never
  editing or replacing anything the parent already exposes.
- Shares state with the parent WITHOUT owning any `dependencies`/`resources`
  of its own — each sub-app reads an already-installed module-level
  singleton the parent's own `setup()` wires (e.g. `getTriggersAggregator()`),
  so composing more sub-apps costs nothing extra in resource-resolution
  complexity. A sub-app needing real DI-managed state instead shares it via
  the parent's own `uses`/root resources, memoized the same way any two apps
  sharing a resource already are.

**Activation — always together, never separately**:
`getAdminHubSubApps()`/`getLocalAdminSubApps()`
(`src/modules/admin-hub-app.ts`/`src/modules/local-admin-app.ts`) each filter an
`Array<{ factory: () => ZanixAppDefinition, enabled: (...) => boolean }>` table down
to the currently-enabled sub-apps, composed via ONE call:

```ts
await activateApps([defineAdminHubApp(options), ...getAdminHubSubApps(options)])
```

`getAdminHubSubApps(options: AdminHubSubAppOptions = {})` takes the SAME
`{ triggers, templates, dlq }` subset already passed to `defineAdminHubApp(options)` —
pass the identical object to both calls (`start.ts`'s own `startSequence` does exactly
this), never a bare `getAdminHubSubApps()`, or a sub-app whose REST controller was
disabled composes anyway. `getLocalAdminSubApps()` stays zero-arg on the local side —
it reads the deployment's own env vars directly, via the same `is<X>ResourceEnabled()`
functions `defineAdminMetadata()` (`metadata.ts`) already gates REST on (see below).

`ZanixAdminHub.start()`/`@zanix/core`'s own `admin: true` option already do
this internally — an author consuming this package through either of those
never calls `getAdminHubSubApps()`/`getLocalAdminSubApps()` directly. It matters for
anyone extending THIS package itself, or building their own package that wants
the same shape.

### Adding a third sub-app — the real, repeatable template

1. Write a new `defineXSubApp(): ZanixAppDefinition` factory — own `name`,
   `routes: false`, its own `operations`/`mcp`, reading whatever shared
   state it needs from an already-installed singleton (or the parent's own
   shared resources).
2. Add `{ factory: defineXSubApp, enabled: ... }` to `HUB_SUB_APP_ENTRIES`/
   `LOCAL_SUB_APP_ENTRIES` (or your own package's equivalent table) — never a bare
   factory list, and never by editing the parent app's own manifest body. `enabled`
   MUST read the exact same signal that resource's REST controller is already gated
   by in `metadata.ts` — see "REST and operations/mcp share ONE gate" below for why
   this isn't a style preference, and which signal shape
   (`is<X>ResourceEnabled()` vs. `options.<x> !== false`) applies to which side.
3. Nothing else changes — `getAdminHubSubApps()`/`getLocalAdminSubApps()`'s callers
   keep working unmodified, since they only ever iterate the filtered list, never a
   fixed arity.

**Deliberately NOT a generic "extension registry"** with its own
install/uninstall lifecycle — it's a plain array of factory functions,
composed through `activateApps()`'s own existing "list of independent
apps" contract. No new primitive was introduced to make this work; it's
the same composition mechanism every Zanix App already uses, applied to a
package's own sub-apps instead of to apps two different teams own.

## A full domain (like Triggers) is FOUR separate pieces, not one task

Confirmed real: building "the DLQ piece" for one of these took three
separate tasks before genuine end-to-end parity with Triggers/Templates was
reached — each piece is a structurally different mechanism, owned by a
different skill, and no single task/agent builds all four:

| Piece | What it is | Owner |
| --- | --- | --- |
| Local sub-app (`operations`/`mcp`) | `defineLocalXApp()`, `resolveTarget`-based operations | `admin-builder`'s own 3-step template |
| Hub sub-app (`operations`/`mcp`) | `defineHubXApp()` | `admin-builder`'s own 3-step template — only once the aggregator row below exists |
| REST + Discovery route wiring | `metadata.ts`'s own `AdminMetadataModuleEntry` table (`registerAdminMetadataModules`'s loop — `createXAdminController` inside `ProgramModule.defineApplication`, `ProgramModule.defineDiscovery('x', createXDiscoveryProvider(), ...)`, one entry per resource) | **Not** the 3-step template (`routes: false` is the opposite of what this wires) — a parallel, real table pattern in the same file (mirrors `admin-hub-app.ts`'s own `AdminHubModuleEntry` table), in scope for `admin-builder` on a case-by-case judgment call until this gets its own explicit skill/carve-out |
| Aggregator + admin client (hub fan-out) | `XAggregator` (`ServiceRegistry`-driven remote fan-out), `XAdminClient` | `admin-triggers-aggregator`/`admin-templates-api`-shaped territory — real, substantial new logic, never a template application |

**A fifth thing to check, not a fifth piece but easy to half-do**: a new
resource's options need threading through BOTH `defineAdminHubApp`'s own
`AdminHubAppOptions` AND `ZanixAdminHub.start()`'s own `StartOptions` —
these are two separate option surfaces (`start.ts`'s convenience entrypoint
composes `defineAdminHubApp` underneath, but doesn't automatically inherit
new fields added to one without also adding them to the other). Confirmed
real: DLQ's own options were correctly added to both in the same task once
this was noticed by reading `admin-hub-app.ts`/`start.ts` in full — adding
a field to only one would leave `ZanixAdminHub.start()` (the primary
documented entrypoint) unable to skip the new route the way it already can
skip `triggers`/`templates`, a real, silent asymmetry.

## Before building a new domain at all — which real shape fits?

**This is Gate 1 — run it before writing any code**, separate from and
earlier than the four-piece status report below (Gate 2), which applies
after one piece of an already-shaped domain gets built. Don't conflate the
two: Gate 1 decides the shape; Gate 2 reports progress on a shape already
decided.

The four-piece table above is ONE of two genuinely different domain shapes
this package has shipped **so far** — both real, both already shipped, not a
"simple version" and a "full version" of the same thing. Treat this as the
catalog of shapes confirmed to exist today, not a closed claim that no third
shape is possible — a future domain's real answers to the three questions
below could genuinely fit neither column.

| | **Proxy/aggregate** (Triggers, DLQ) | **Centrally-owned** (Templates) |
| --- | --- | --- |
| Who holds the real data | Each business service, its own collection | The hub itself, directly |
| Hub's own role | Fans out to N services via `ServiceRegistry` (`XAggregator`) | Owns and serves the data itself — no aggregator class at all |
| Extra reconciliation? | None needed | A `sync` extension (`POST /templates/sync`) keeping a service's code-defined content in step with the hub's own copy |
| Storage "modes"? | No — the data only ever lives one place (that service's own collection) | Yes — content can legitimately live in code, a service's own DB, or centrally at the hub, because it's the SAME shareable content in three forms |
| Skill covering the shape | `admin-triggers-aggregator` (mechanics) + this skill's four-piece table (what to build) | `admin-templates-api` (mechanics only — documents Templates' own existing behavior, not a repeatable "build a new one" recipe yet) |

**Ask these before picking a shape for a new domain — don't default to
mirroring whichever one you read most recently:**

1. **Does more than one service ever need to see the SAME instance of this
   data, or does each service's own copy naturally stand alone?** Triggers/
   DLQ: a service's own triggers/dead-letters are inherently that service's
   own — nothing about them is shared content. Templates: the same rendered
   email/SMS content is often genuinely the same across services (shared
   branding, a company-wide notice) — that's what makes "one central copy,
   others reference it" a coherent idea in the first place. If nothing
   about the new domain's data is naturally shareable across services,
   proxy/aggregate is very likely right regardless of how tempting central
   ownership looks — don't introduce sharing a domain never asked for.
2. **Could this data legitimately be authored in code at all** (a
   compile-time default a human writes), not just persisted at runtime?
   Templates' code-registry mode exists because a template genuinely has a
   sensible "ships in the package" default. Triggers/DLQ entries are
   runtime facts (a configured reaction, a failure record) — there's no
   "code" version of either, so a code-mode isn't a real option regardless
   of which shape is chosen. A domain with no code-authorable form skips
   this axis entirely, not adapt it partially.
3. **Does staying in sync across "where it lives" need active
   reconciliation, or is one-copy-per-service self-consistent by
   construction?** Templates needs `sync` because code, a service's DB, and
   the hub's DB can each drift from the others independently. A
   proxy/aggregate domain has nothing to reconcile — there IS only the one
   copy per service, so there's nothing to fall out of sync with.

**If the answers land on proxy/aggregate**: follow the four-piece table
above exactly, `admin-triggers-aggregator`/DLQ's own precedent files.

**If the answers land on centrally-owned (Templates' shape)**: this is
real, in-scope work for `admin-builder`, but the skill only has ONE worked
example (Templates itself) and it predates this decision framework — no
second, independently-built precedent exists yet the way Triggers/DLQ now
cross-validate each other for the proxy/aggregate shape. Treat
`admin-templates-api`'s documented mechanics as the template to mirror
(two-controller composition — CRUD owned by the data package, `sync` owned
by this package; the Discovery-resource preference order; the
concurrency-safe upsert semantics), read the real
`templates-sync.handler.ts`/`templates-sync.ts`/`templates.client.ts` files
directly rather than assuming this table's summary is complete, and expect
to make more real judgment calls than the proxy/aggregate shape needs
(there's no equivalent of "which methods stay off the admin surface" — the
open questions here are more about "does this domain's data genuinely need
all three modes, or just central+code with no per-service DB at all,"
which the skill doesn't have a second data point to generalize from yet).

**Before concluding a domain doesn't fit either shape, check whether it's
actually two domains wearing one name.** The three questions above assume
one answer per domain — but a single admin request can genuinely bundle two
logically distinct pieces of data with two different real shapes.
Confirmed real (from testing this exact framework): "service health
thresholds" splits cleanly into a per-service override (Triggers/DLQ-shaped
— that service's own fact, nothing to reconcile) and an org-wide default
(Templates-shaped — has a code mode, wants central visibility); each piece
answers all three questions cleanly **on its own**, only the seam between
them ("does the hub need to know when a service deviates from the
default?") stays genuinely open. Before reporting "doesn't fit either
shape," re-run the three questions separately for each logically distinct
piece the request actually describes — if every piece lands cleanly on a
known column, this isn't a third-shape case at all, it's two known shapes
stitched by an ambiguous seam, and the sharper, more answerable question to
raise is about that seam specifically ("does X need to reconcile against Y,
yes or no?"), not a generic "which shape is this?" Only treat it as a
genuine third-shape candidate when a single, non-decomposable piece of data
still doesn't fit either column after this check.

**If, after that check, the answers genuinely don't cleanly land on either
column** — a single non-decomposable piece of data whose question-1/2/3
answers point at neither shape, or point at a kind of coupling these three
questions don't probe at all — **don't force it into the nearer-looking
shape just because it's the only two on record.** Say so explicitly: name
which answers didn't fit (or, per the decomposition check above, name the
pieces and the specific seam question), and flag the domain as needing real
design work rather than silently mirroring Triggers/DLQ or Templates and
hoping the mismatch doesn't matter. This is exactly the kind of judgment
call worth raising as a clarifying question before building, rather than
deciding unilaterally — `admin-builder` should ask, not guess. A genuinely
new third shape, once built and validated, belongs back in this table as
its own column, the same way DLQ's build cross-validated proxy/aggregate as
a repeatable shape rather than a Triggers-only one-off.

**Gate 2 — whoever builds any ONE of these four must report the other
three's status explicitly**, not just what was built — "X exists, Y exists, Z is still
missing because <reason>" — even when building the missing pieces is out of
scope for the current task. The orchestrator/human dispatching the work
often won't have this full map memorized (confirmed real: it took three
separate dispatches to discover all four pieces for DLQ, each one
"finishing," rediscovering the next gap only afterward) — the report is
where this gets caught, not by whoever is dispatching already knowing to
ask.

## On-by-default vs. opt-in for a new `metadata.ts` route — the real discriminator

`metadata.ts`'s existing blocks aren't arbitrarily different: Triggers is
on-by-default-unless-disabled (`isTriggersModelDisabled()`), Templates is
opt-in (`templatesBackendMode() === 'local'`, i.e. `TEMPLATES_BACKEND=local`
must be explicitly selected). **The rule that
decides which shape a new resource gets**: is the underlying model
auto-registered as a side effect of something that already always runs
(Triggers' collection is resolved unconditionally inside
`ZanixMongoConnector`'s own constructor, in `@zanix/datamaster`) — then
default the route on, gated only by an explicit opt-out env var? Or does it
require a standalone bootstrap call a host must make explicitly (DLQ's
`registerDLQModel()`, Templates' own setup) — then the route must default
OFF, gated by that same opt-in env var, since defaulting a route on in
front of a possibly-never-registered model fails at request time, not at
boot. Verify which shape actually applies by reading the real
registration path in the sibling package (not by guessing from how similar
the domain "feels" to an existing one) before choosing.

## REST and operations/mcp share ONE gate — `is<X>ResourceEnabled()`

Every resource `metadata.ts` gates for REST/Discovery
(`isTriggersResourceEnabled()`/`isTemplatesResourceEnabled()`/`isDlqResourceEnabled()`,
`admin-resource-gates.ts`) gates that SAME resource's operations/mcp sub-app the same
way — `LOCAL_SUB_APP_ENTRIES` calls the identical functions. This wasn't always true:
`getLocalAdminSubApps()`/`getAdminHubSubApps()` used to compose every sub-app
unconditionally, regardless of whether that resource's REST controller was even
registered — a reachable-but-broken "ghost" operations/mcp surface in any deployment
that never configured `templates`/`dlq`. Fixed 2026-08-22 on both the local side (env
var-driven) and the hub side (`AdminHubSubAppOptions`-driven) — see
`local-admin-app.ts`/`admin-hub-app.ts`'s own doc comments for the full before/after.

`admin-resource-gates.ts` is a thin delegation layer, not the logic owner — it
re-exports the real owner packages' `is<X>ResourceEnabled()` functions verbatim. See
`zanix-envvar-conventions`'s own "companion `is<X>ResourceEnabled()` boolean" section
for the general convention, which package owns which function, and when to add one for
a NEW env-var-gated resource — not repeated here; this skill only covers the
consequence specific to this package: a new resource's `enabled` predicate (its
`metadata.ts` entry AND its sub-app entry) must call the SAME function, never two
independent derivations.

The hub side gates on an explicit `options.<x> !== false` argument instead
(`AdminHubSubAppOptions`, `getAdminHubSubApps(options)`), never an `is<X>ResourceEnabled()`
call — the hub is a different process from the business service whose env vars it has
no way to read; its `enabled` predicate reads the exact same option
`registerAdminHubModules` already gates the REST controller by. Both sides apply the
same principle (one decision point per resource, shared by REST and operations/mcp) to
two different signal shapes (env var vs. constructor option) — don't force the hub
side onto the env-var shape, and don't assume the local side ever takes an options
argument the way the hub side does.

## Checklist before composing or extending this package

- [ ] Is `ZanixAdminHub.start()` the right choice for this deployment, or
      does it genuinely need direct `defineAdminHubApp()`/
      `defineLocalAdminApp()` composition alongside other apps?
- [ ] Does a new sub-app follow the 3-step template exactly — own `name`,
      `routes: false`, added to the `{factory, enabled}` entries table with
      `enabled` reading the SAME signal that resource's REST controller is
      gated by, nothing else touched?
- [ ] Does the new sub-app read shared state from an already-installed
      singleton or the parent's own resources, rather than declaring its
      own `dependencies`/`resources`?
- [ ] Is this genuinely an Extension (adding new capability) rather than an
      Override (replacing existing behavior) — the wrong mechanism for the
      latter?
- [ ] **Does a new hub-side REST controller (like Triggers/DLQ) get added to
      `start.test.ts`'s own real-`ZanixAdminHub.start()`-plus-real-`fetch`
      check, not just its own isolated unit test?** Confirmed real gap, not
      hypothetical: `start.test.ts` already existed as the one place that
      boots the real hub composition and does a real HTTP fetch against
      `/triggers`/`/templates` at the `admin-hub` Application level (401
      behind auth) — when DLQ's hub controller (`createDlqController`) was
      added, this file was never updated to also fetch `/dlq`, so DLQ's own
      real-HTTP-reachability coverage silently fell BELOW Triggers'/
      Templates' existing level rather than merely matching a shared,
      preexisting gap. Fixed by adding the `/dlq/list` fetch alongside the
      existing two. `dlq.aggregator.test.ts`/`dlq.handler.test.ts`
      (isolated, in-process controller tests) don't substitute for this —
      they never exercise the real `AuthTokenValidation` guard over a real
      socket the way `start.test.ts` does. `@zanix/datamaster/dlq-api` is
      published as of `@zanix/datamaster@1.5.0`, which `admin`'s own
      `deno.lock` pins — `metadata.ts` imports `createDlqAdminController`
      from it unconditionally, and `ZanixAdminHub.start()`'s default path
      (dlq on by default) is runtime-verified end-to-end by
      `start.test.ts`'s real `fetch('/dlq/list')` check.
- [ ] **Before building a new sub-app at all**, does its target domain
      already have what each side of the 3-step template actually needs to
      point at — not just a raw provider/connector? Local side: a
      DI-managed, admin-facing Interactor/Service class in the sibling
      package (shaped like `TriggersAdminService`/`DlqAdminService`,
      `resolveTarget`-able) — not bare `<X>Provider`-shaped raw provider
      access. Hub side: a real admin REST controller exported from that
      package (like `triggers-api`'s `createTriggersAdminController` or
      `dlq-api`'s `createDlqAdminController`) for the aggregator to proxy
      to. Don't work around a missing one by reaching into the raw
      provider directly from an operation handler — verify what actually
      exists for real (grep the sibling repo, check its `deno.jsonc`
      exports) before concluding either way, never assumed from the
      domain name alone; DLQ was the confirmed real example of neither
      side existing yet, until `@zanix/datamaster`'s `dlq-api` landed (see
      the next checklist item — DLQ is now the full-parity precedent, not
      the missing-prerequisite example). **Verified in the sibling repo's
      own source ≠ verified as published** — a symbol can be real in that
      repo's uncommitted working tree while the currently-published JSR
      version still lacks it entirely (query the real registry, e.g.
      `https://jsr.io/@zanix/<pkg>/meta.json`'s `latest`, then that
      version's own `exports` map — don't stop at confirming the source
      exists), so a new local-side file can be correct and still fail
      `deno check` until the sibling package is actually republished — a
      known, expected class of gap while a migration like this is in
      flight, not a bug in the new code.
- [ ] **A sub-app is not required to exist on both sides.** Build only the
      side whose real prerequisite actually exists — a local sub-app needs
      just the DI-managed service class; a hub sub-app additionally needs a
      real aggregator (`ServiceRegistry`-driven remote fan-out, mirroring
      `TriggersAggregator`) and admin client for that domain, which is
      itself real, substantial new aggregation-layer work, not a
      3-step-template application. Building the local side alone and
      leaving the hub side unbuilt (with the gap flagged explicitly, in
      both the code's own JSDoc and the README) is a valid, complete
      outcome for a domain still missing its hub-side prerequisite — that
      was DLQ's own interim state before `DlqAggregator`/`DlqAdminClient`
      landed; DLQ now HAS both `defineLocalDlqApp()` and its
      `defineHubDlqApp()` counterpart, the same full-parity shape
      Triggers/Templates already have, so it's no longer the worked example
      of a domain missing its hub side — treat it as full-parity precedent
      instead, and look for a domain still missing its aggregator/admin
      client if you need a live example of the partial-build case.

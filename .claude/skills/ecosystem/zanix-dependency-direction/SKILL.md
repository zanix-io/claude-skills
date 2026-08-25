---
name: zanix-dependency-direction
description: Dependency-direction and package-tier rules across the Zanix ecosystem (@zanix/server, @zanix/datamaster, @zanix/auth, @zanix/notifications, @zanix/asyncmq, @zanix/admin, @zanix/core, plus @zanix/app/@zanix/space as consumers of the same tiers). Use before adding any new cross-package import, designing a registry that crosses package boundaries, or deciding which tier a brand-new package belongs to.
---

This skill exists to stop the same dependency-direction mistakes from being
reintroduced across sessions — either a design round caught one before it
shipped, or it shipped as a real bug that a later round found and fixed. File:line
references point at the actual precedent in this monorepo
(`~/Documents/Development/ZanixLibraries/{server,admin,core,notifications,
datamaster,auth}`) — read the real code there before assuming this summary is
still accurate; it will drift as the packages evolve.

For rules about designing a *new mechanism inside* `@zanix/server` itself
(composition/activation layering, the core-slot registry, registration-function
shape), see `zanix-server-internals`. For where an HTTP controller over a
package's own data belongs, see `zanix-local-api-vs-aggregator` — that rule is
one specific consequence of the dependency direction described here, split out
because it's referenced independently by several packages' own docs.

## Golden rule (token savings)

- **Re-verifying the tier list is one grep loop (see below), not a per-package
  investigation.** Run it once for the packages actually involved in the
  import you're checking — don't re-derive the whole hierarchy from scratch
  or read every package's full `deno.json` by hand.
- A change that clearly points downward on the tier list (the common case)
  needs a one-line confirmation, not a worked-example write-up — save the
  worked-example depth for a genuine same-tier or upward case, which is
  exactly what needs the inversion pattern explained.

## The general rule

A package may depend on any capability lower than it in the hierarchy below. It
must never depend sideways (a peer at its own tier) or upward (a tier above it).
This is not "no package may depend on another" — a modular ecosystem needs
composition, and several of these dependencies (e.g. `notifications ->
datamaster`) are real, intentional, and necessary. The rule is about
*direction*, not *whether*.

This tier list was built by reading each package's real `deno.json`/`deno.jsonc`
import map and grepping actual (non-comment) imports across the whole monorepo —
not assumed from an earlier summary. Re-verify it the same way before relying on
it if enough time has passed that it might have drifted:

```
for pkg in server datamaster notifications asyncmq auth admin core; do
  grep -oE '"@zanix/[a-z]+(/[a-z]+)?":\s*"[^"]*"' "$pkg/deno.json"* 2>/dev/null
done
```

## Tier hierarchy

1. **Kernel — `@zanix/server`.** Zero dependencies on any other Zanix package.
   Owns DI, decorators, Application/Runtime/route composition, and the open
   slot-registry mechanism (`registerCoreProviderSlot`/`registerCoreConnectorSlot`
   — see `zanix-server-internals`). This is the *reference instance* of "a lower
   layer offers a registry instead of importing upward": `@zanix/server` never
   imports `@zanix/auth`/`@zanix/datamaster`/anything else to know their concrete
   types — it just exposes a slot mechanism those packages opt into from their own
   `/core` entrypoints.
2. **Foundational infrastructure — `@zanix/datamaster`.** Depends only on
   `server`. This is NOT a peer of `auth`/`notifications`/`asyncmq` — it sits
   structurally *below* them, because it provides the thing every domain
   capability that needs persistence is built on: connectors (Mongo/SQLite),
   model/schema registration (`registerModel`), and — importantly — it also
   hosts **its own open registries** for the same reason `server` hosts the slot
   registry: `TriggerActionJobsContainer`/`registerTriggerActionJob`
   (`datamaster/src/modules/program/metadata/trigger-action-jobs.ts`, built on
   `@zanix/server`'s own `ProgramContainer` base class — reused plumbing, not
   reinvented) lets a domain package register a job descriptor without
   datamaster ever needing to know who registered it or why. A package at this
   tier may depend downward on `server`; it must never depend on
   `auth`/`notifications`/`asyncmq`/`admin`/`core` (verified: zero real imports
   of any of them anywhere in `datamaster/src`).
3. **Domain infrastructure capabilities — `@zanix/auth`, `@zanix/notifications`,
   `@zanix/asyncmq`.** Each depends on `server` (kernel), and **may depend
   downward on `datamaster`** when it genuinely needs persistence/connectors/schema
   registration for its own domain data — this is a *valid direct dependency*,
   not something to route around. The real, verified example: `notifications`
   imports `ZanixMongoConnector`, `Model`, `MongoModelDefinition`, and calls
   `registerModel` from `@zanix/datamaster` directly
   (`notifications/src/modules/templates/{provider,core}.ts`,
   `db/{schema,templates.repository,local-backend}.ts`) to persist its own
   templates collection — concrete types included, because `notifications`
   structurally needs Mongo specifically, not an abstraction over "some
   database." **What these three must never do is depend on each other
   sideways** — no `asyncmq -> auth`, no `auth -> notifications` (verified:
   zero such imports exist in either direction). **One real, confirmed
   exception, found by a `dependency-direction-sweep` run and independently
   re-verified against real source before trusting it (now fixed — see
   below)**: `notifications -> auth`, real via `notifications/deno.jsonc`'s
   own `imports`, isolated (as of the fix) to a single adapter module,
   `notifications/src/modules/templates/db/remote-backend-auth.ts:1,8`
   (`createServiceAuthClient`/`resolveServiceAssertionKeyId`/
   `resolveServiceAssertionPrivateKey`/`ServiceAuthClientOptions`) — not a
   comment/type-only reference — `RemoteTemplateBackend` (the "remote-only templates"/Mode C
   path, see `notifications/docs/templates.md`) authenticates its own
   service-to-service fetch against `@zanix/auth`'s `type: 'api'` contract;
   `auth`'s own source already documents this coupling from its side too
   (`auth/src/utils/sessions/service-exchange.ts`). Confirmed
   one-directional — `auth` never imports `notifications` back, so this is a
   real sideways dependency, not a cycle. **Fixed**, applying "Worked example
   3" below (the constructor-injected type-only seam + separate adapter
   module `TriggersClientFactory`/`admin/src/modules/registry/auth.ts`
   already use for the identical shape): the real `@zanix/auth` import now
   lives isolated in a NEW adapter module,
   `notifications/src/modules/templates/db/remote-backend-auth.ts` (its own
   module doc names it as the one file allowed to do this), exporting
   `createRemoteTemplateAuthClient` (wraps `createServiceAuthClient`,
   module-level build-once cache) and `assertServiceAssertionKeyResolvable`
   (wraps `resolveServiceAssertionKeyId`/`resolveServiceAssertionPrivateKey`).
   `remote-backend.ts:12-25` now defines only the type-only seam
   (`ServiceAuthClient = (targetServiceId, exchangeUrl) => Promise<Record<string,string>>`);
   `RemoteTemplateBackendConfig.authClient` (`remote-backend.ts:129`) accepts
   an already-built one via constructor injection, defaulting to `undefined`
   (no auth header at all) when omitted — the same "safe, unauthenticated
   default" `TriggersClientFactory` uses. `provider.ts` no longer imports
   `@zanix/auth` at all; `assertTemplatesBackendConfigValid()` (renamed from
   `assertTemplatesConfigNotConflicting()` in the later `TEMPLATES_BACKEND`
   selector rework — see `zanix-envvar-conventions` — same adapter-isolation
   shape unaffected by that rename) and `#backend()` both call the new
   adapter instead. **The one real
   complication this case had that Worked example 3 doesn't cover
   explicitly**: unlike `admin`'s `TriggersAggregator` (which has an external
   composing layer, `ZanixAdminHub.start({ auth })`, living in a different
   package entirely), `notifications` has NO external composing layer for
   Mode C — `TemplateProvider.#backend()` is the ONLY place
   `RemoteTemplateBackend` is ever constructed from env vars, and Mode C must
   stay self-configuring from env vars alone with zero app-side wiring. The
   resolution: `provider.ts` itself plays the composing-layer role Worked
   example 3 describes as a separate package — it's simultaneously "the class
   that needs the capability" (for `assertTemplatesBackendConfigValid()`'s
   validation) and "the composing layer" (for `#backend()`'s construction of
   `RemoteTemplateBackend`), which is why it's the one file besides the
   adapter that imports the adapter's real functions. A second complication:
   `RemoteTemplateBackend`/`RemoteTemplateBackendConfig` are themselves public
   package exports (`mod.ts`) with a real test
   (`notifications/src/@tests/integration/remote-template-backend.test.ts`)
   constructing `RemoteTemplateBackend` directly with a full
   `{serviceId, privateKey}` auth config, bypassing `TemplateProvider`
   entirely and expecting the real sign+exchange to happen — so the seam
   couldn't just default to "always unauthenticated, only `provider.ts` ever
   wires real auth"; any direct caller (including that test) has to build its
   own `ServiceAuthClient` via the adapter's `createRemoteTemplateAuthClient`
   and inject it explicitly, same as `provider.ts` does. **Not
   `@zanix/utils` promotion** — service-credential signing/exchange is
   genuinely `@zanix/auth`'s own domain, not a neutral utility (see Worked
   example 3's own ownership-discipline note); a fix that relocates this
   logic to `@zanix/utils` trades one violation for another, don't converge
   on it just because a `@zanix/utils` promotion precedent exists for other,
   genuinely-neutral cases. Re-verify these file:line references against the
   real repo before trusting them, same as every other one in this skill, in
   case they've drifted since. When one of these three packages needs a
   capability that's really another sibling's domain, that's a signal for
   inversion via a registry, never a direct import.
4. **Applications — `@zanix/admin`.** Depends on any tier-0-through-2 package it
   needs, **including their concrete types** (`ZanixMongoConnector`, not just an
   abstract connector contract) — because it's a product built on top of the
   stack, not a generic capability other packages might build on top of in turn.
   Composed into `@zanix/core` via its own bespoke mechanism
   (`defineAdminMetadata`), never through the slot registry.
5. **Orchestrator — `@zanix/core`.** Depends on everything. This is the *only*
   tier allowed to import two siblings from any lower tier (two tier-2 packages,
   a tier-1 and a tier-2, etc.) and wire them into each other directly —
   composing is its entire job. This is what makes the registry-inversion
   pattern below work at all: a lower package publishes into a registry, and
   `core` (never the publisher) is the one place that both drains it and holds
   the sideways import the publisher was never allowed to hold itself.

`@zanix/app` and `@zanix/space` sit alongside this hierarchy as consumers, not
as new tiers of their own: `@zanix/app`'s `.`/`./runtime`/`./core` entrypoints
form their own one-directional DAG on top of `server` (`@zanix/server` never
imports anything from `@zanix/app`), and `@zanix/space` is itself a Zanix App —
activated by `@zanix/server`, never the other way around. Resource types like
`'mongo'`/`'redis'` in an app manifest resolve directly to `@zanix/datamaster`'s
real connectors, never a re-declared shape — the same "depend downward on the
concrete type" allowance tier 2/4 packages already have.

## When a direct dependency is valid vs. when inversion is required

- **Valid direct dependency**: depending downward on a lower tier for a
  capability that tier exists to provide (persistence, connectors, kernel DI
  primitives). `notifications -> datamaster` for Mongo persistence is this —
  don't invent an abstraction layer to avoid it; the dependency is honest and
  the tier ordering already makes it safe (datamaster can never depend back).
  `asyncmq -> datamaster` for `DlqProvider` (the Dead Letter Queue's persistence)
  is the same shape, confirmed in `asyncmq/docs/dlq-reprocessing.md`: a
  dedicated subpath (`@zanix/asyncmq/dlq`) so importing the rest of `asyncmq`
  never pulls in `datamaster`'s module graph for apps that don't use the
  feature.
- **Requires inversion (registry/interface/event), never a direct import**: two
  packages at the *same* tier need to reach each other, or a *lower* tier needs
  a capability that conceptually belongs to a *higher* one. Both worked examples
  below are drawn from a real precedent already in this monorepo, not
  hypothetical:

**"Inversion" doesn't always mean the dependency disappears — sometimes the
correct end state is confinement, not elimination, and which one applies
depends entirely on which worked example the case matches.** Worked examples
1/2's registry-drain shape genuinely eliminates the dependency from the
publishing package — `notifications`/`asyncmq` never import each other at
all, `core` alone holds both sides. A genuinely neutral utility (the
`@zanix/utils` promotion case) also genuinely eliminates it — the logic
moves to a tier both sides already depend on, so neither imports the other
anymore. **Worked example 3 is different on purpose**: when the capability
is synchronous AND genuinely belongs to the owning package's own domain
(not neutral, not deferrable), the sideways import cannot be eliminated
without either duplicating that domain logic or breaking the feature that
needs it — both worse than the sideways import itself. The correct, FINAL
end state there is confinement: exactly one adapter module keeps the real
import, everything else in the consuming package is import-free (not even
`import type`, per the seam-type-not-borrowed-type design — see Worked
example 3's own note on why the seam defines its own shape rather than
importing the owner's type). Don't chase further "elimination" once a
Worked-example-3 fix is confined to its one adapter module — that adapter
IS the fix, not a partial step toward removing the import entirely.

**Worked example 1 (real today) — a domain capability's job needs to run on
`asyncmq`, without `notifications -> asyncmq`:** `notifications`'s `mail`
trigger action needs to become a real `asyncmq` job eventually, but
`notifications` must never import `asyncmq` (same-tier sideways). Fixed by
routing through the tier BELOW both of them: `notifications` registers a
transport-agnostic job descriptor — handler typed against a minimal
`{ providers: ZanixProvidersGetter }` context, not `asyncmq`'s own `Job` type —
into `datamaster`'s `TriggerActionJobsContainer`
(`notifications/src/modules/providers/trigger-mail.core.ts`'s
`registerMailTriggerJob`, loaded from notifications' own `/core` entrypoint,
same "plain re-callable registration function" rule as `zanix-server-internals`
describes). `@zanix/core` — the orchestrator, the only tier allowed to hold both
imports — drains that registry and performs the actual `@zanix/asyncmq`
`registerJob` call (`core/src/utils/metadata.ts`'s
`registerPendingTriggerActionJobs`). Generalize this: **when a future scenario
needs `notifications` to expose jobs that `asyncmq` runs, and/or `asyncmq`
itself gains a `datamaster` dependency for its own persistence needs**, the
shape stays the same — register a descriptor into a registry hosted at the
lowest tier both sides already legitimately touch (or a new dedicated registry
there, extending `ProgramContainer` the same way `TriggerActionJobsContainer`
does), and let `core` be the only place that ever imports both sides to wire
them together. Neither `notifications -> asyncmq` nor `asyncmq -> notifications`
should ever appear in a `deno.json`.

**Worked example 2 (hypothetical, same shape) — `datamaster` wants to expose a
job or another capability that conceptually belongs to a higher tier:** if
`datamaster` (tier 1) needed to define a job, it must NOT import
`@zanix/asyncmq` (tier 2) to get `Job`/`registerJob` — that would be a lower
tier depending on a higher one, exactly the direction this whole section
forbids, and it would make `datamaster` fragile to `asyncmq`-specific churn for
a capability datamaster doesn't own the runtime semantics of anyway. Instead,
apply the same registry-hosting pattern datamaster ALREADY uses for
`TriggerActionJobsContainer` in the other direction: define the job's logic as a
transport-agnostic descriptor (same `{ providers }`-typed handler shape, no
`asyncmq` import) and expose it through `datamaster`'s *own* open registry (or
extend the existing trigger-action one, if it's the same shape of capability)
for a *higher* tier — `core`, or `asyncmq` itself reaching down to register
against it — to drain and activate. The mirror precedent already proves this
works both directions: `server` (tier 0) exposes `registerCoreProviderSlot` for
tier-1/2 packages to opt into, rather than `server` importing
`datamaster`/`auth` to know their concrete provider types upfront. **Note this
was deliberately NOT reused for the DLQ case above** (`datamaster/docs/dlq.md`
explains why): a registry-drain inversion solves a *lateral* dependency problem
between same-tier siblings; `asyncmq -> datamaster` for `DlqProvider` has no such
problem, since it's already a valid downward dependency — don't add an inversion
layer where a plain direct import is already correct.

**Worked example 3 (real today) — a sibling's capability is needed
SYNCHRONOUSLY, at call time, not as a deferrable job**: worked examples 1/2's
registry-drain shape only fits when the need can wait for a later drain step
(`core` draining a job descriptor). It does NOT fit when the consumer needs
the sibling's capability invoked inline, in the same call — the shape
`admin`'s own `TriggersAggregator` uses for per-service auth is the real
precedent for THIS case, not the registry pattern: `TriggersAggregator`
(`admin/src/modules/triggers/triggers.aggregator.ts`) needs to attach a
service credential to every outbound request, but never imports `@zanix/auth`
itself — it defines a type-only seam (`TriggersClientFactory = (service) =>
Client | Promise<Client>`), injected via constructor, defaulting to an
unauthenticated client. The one real `@zanix/auth` import
(`createServiceAuthClient`) lives isolated in a separate adapter module
(`admin/src/modules/registry/auth.ts`'s `createServiceRegistryAuthHeaders`)
that only a caller wiring real auth ever reaches — `ZanixAdminHub.start({
auth })` connects the two. Generalize this: when package X (same tier or
higher than the capability's owner) needs to invoke that capability inline
rather than defer it, X defines only the seam's TYPE, accepts it via
constructor/parameter injection with a safe default, and the real import to
the owning package lives in a separate adapter module that only the
composing layer (here, `admin` itself, composing its own sibling) ever
imports both sides to connect — never inside the class/function that needs
the capability. **Ownership discipline still applies**: the adapter imports
the owning package's real logic (`createServiceAuthClient`) rather than
duplicating or relocating it — moving that logic itself to a lower/neutral
tier (e.g. `@zanix/utils`) is the WRONG fix when the logic is genuinely the
owning package's own domain (service-credential signing/exchange is
`@zanix/auth`'s domain, not a generic utility) — that trades one violation
(sideways ownership of the capability) for another (stripping a package of
its own real domain logic). Reach for `@zanix/utils` promotion only when the
thing being shared is genuinely a neutral utility with no real owner
(`SESSION_COOKIE_ATTRIBUTES`, `confinePath` — real, already-promoted
precedents), never as a default move to resolve *any* sideways dependency.

**Two checks before assuming an external composing layer will wire the real
adapter in, confirmed necessary by the `notifications -> auth` fix below**:
(1) does an external, higher-tier package actually compose this class with
the capability's owner (`admin`'s `ZanixAdminHub.start({ auth })` for
`TriggersAggregator`), or is the class entirely self-configuring from env
vars with no such call site — if the latter, the class's OWN construction
site (wherever it builds itself from config) has to play the composing-layer
role itself, importing the adapter directly, rather than assuming some other
package will inject the real seam; (2) is the class a public package export
with any test (`grep` for `new <Class>(` across `src/@tests/**` too, not
just `src/**`) that constructs it directly and expects the real capability
to run — if so, the seam's default can't just be "always unauthenticated,
only the internal composing layer ever wires it," since a direct external
caller (including that test) needs the same adapter-built value to inject
itself. Skipping either check produces a seam that looks complete but is
either unreachable in practice (nothing ever supplies a real adapter) or
breaks a real, existing direct-construction call site the moment the fix
ships.

## Preventing circular dependencies going forward

Because the tiers above form a strict partial order (0 → 1 → 2 → 3 → 4, plus
same-tier siblings at tier 2 that must never point at each other), a cycle can
only be introduced by one of exactly two mistakes — check for both before adding
any new cross-package import:

1. **A new import that points at an equal-or-higher tier.** Before adding
   `import ... from '@zanix/X'` in package `Y`, place both `X` and `Y` on the
   tier list above. If `X`'s tier number is greater than or equal to `Y`'s (same
   tier counts as a violation, not just strictly higher), it's not a valid
   direct dependency — apply the inversion pattern instead, don't add the import
   anyway "just this once."
2. **A registry consumer that becomes a registry producer for the same
   capability.** The `TriggerActionJobsContainer` pattern stays acyclic only
   because `datamaster` (the registry host) never itself calls
   `.resolve()`/reads its own registry to drive behavior that reaches back into
   the tier that populated it — it's a passive container. If a lower-tier
   registry's *host* package starts reacting to what a higher-tier package
   registered into it (beyond just storing/returning it opaquely), that host has
   effectively started depending on the higher tier's behavior, which is the
   same cycle by a different route. Keep registry hosts passive: store and
   return, never interpret.

When in doubt about which tier a NEW package belongs to, ask: what's the lowest
tier that already provides everything this package structurally needs? That's
its tier — don't place it higher "to be safe," since that only makes its own
future dependents more likely to violate the ordering above by depending on it
for something that didn't need the higher placement at all.

## A second, different-axis cycle: intra-package circular imports with a top-level side effect

Everything above is about the direction of imports BETWEEN packages
(`@zanix/*`). This section is a different axis entirely: a cycle formed by
plain relative or path-aliased imports BETWEEN FILES INSIDE ONE PACKAGE'S
OWN `src/` tree — `dependency-direction-sweep`'s cross-repo graph
structurally cannot see this (it's built only from `@zanix/*` imports).

**The real, built, automated mechanism for this axis is `zanix check-cycles`**
(`@zanix/cli`, `cli/src/commands/check-cycles/`) — a Deno-native command (no
third-party parser; it uses `deno info --json` for the real import graph and
`Deno.lint.runPlugin` for a real AST pass, the latter run as its own `deno
test` subprocess since that API only works in that context) that finds a
real cycle AND checks whether any file in it executes a top-level statement
reading a cross-cycle binding — the same two-part rule this section
describes. Run it directly: `zanix check-cycles --path <package-root>`,
exits non-zero on a confirmed finding. `dependency-direction-sweep` runs
this same command per repo rather than re-deriving the algorithm itself —
see that agent. This skill states the rule; the command IS the mechanism,
not a manual `deno info` trace — reach for the manual trace below only when
the command isn't available in the current context.

**Real precedent (shipped as a bug, then fixed) — `@zanix/notifications`,
`src/modules/email/`:** `defs.ts` imported `SmtpClient` from `connector.ts`;
`connector.ts` imported `getSmtpPool`/`SmtpConnection` from `pool.ts`;
`pool.ts` imported `SMTP_POOL_SIZE_ENV` from `defs.ts` — closing a 3-file
cycle. `defs.ts`'s own top-level call `registerSmtpConnector()` (line 69,
the same "eager module-load registration" idiom
`datamaster/storage/core.ts` and every `datamaster-connector-registration`
backend also use — see that skill) read `SmtpClient.config` while
`connector.ts` was still mid-evaluation, because the real entrypoint
(`connector.ts`, imported directly by `notifications/mod.ts`) hadn't
finished declaring the `SmtpClient` class yet when the cycle looped back
into `defs.ts`. First time `SMTP_HOST`/`SMTP_PORT`/`SMTP_USER`/
`SMTP_PASSWORD` were all set, this threw a real
`ReferenceError: Cannot access 'SmtpClient' before initialization` (a
temporal-dead-zone violation) — not a type error, `deno check` passes this
shape clean. **Fixed** by moving `SMTP_POOL_SIZE_ENV`'s definition into
`pool.ts` (its only real consumer, `getSmtpPool()`) — verified against the
real current source: `defs.ts` does NOT re-export it from there, nothing
else in the package imports it from `defs.ts` either, so no re-export was
needed. This removes the `pool.ts → defs.ts` edge entirely, leaving the
chain one-directional (`defs.ts → connector.ts → pool.ts`, no way back).

**The cycle alone is not the danger — most cycles in this ecosystem are
harmless** (type-only references, or references confined inside a function
body that only ever runs after the whole module graph has finished
loading). **The actual danger is the intersection**: a file inside the
cycle executes something at module TOP LEVEL — a function call, `new X()`,
a decorator application (`Connector(...)(SmtpClient)`), a registration call
— that reads a binding (a class, a value) imported from another file still
inside that same cycle. The eager-registration idiom itself
(`registerXConnector()`/`registerCoreConnectorSlot` running unconditionally
at module load) is a deliberate, repeated, otherwise-safe pattern across
this ecosystem — the idiom is not the bug; pairing it with a same-package
import cycle is.

**Checklist before adding/editing two or more files that import each other
within one package** (file-level version of the checklist below):

- [ ] Run `zanix check-cycles --path <package-root>` (the real, built tool
      — see above). A confirmed finding fails the command outright with the
      exact file:line and cycle; a clean run reports the cycle count found
      (if any) with none of them risky. If the command isn't available in
      the current context, `deno info <entrypoint>` on the package's real
      public entrypoint shows the same graph manually — a cyclic import
      appears marked as already-visited — but confirming the risky
      intersection by hand from there is exactly the error-prone manual
      work the command exists to replace; prefer the command.
- [ ] If a cycle exists, does any file in it execute a top-level statement
      (outside a function/class-method body — a plain `const`/`function`/
      `class` declaration is fine on its own) that reads a binding imported
      from another file in that same cycle? If so, trace the REAL
      entrypoint's actual load order (not just "it compiles") to confirm
      which file's declarations are guaranteed to have already run by the
      time that top-level statement executes — the same trace that found
      the `defs.ts`/`SmtpClient` bug above.
- [ ] Prefer breaking the cycle over reasoning that the current load order
      is "probably fine" — move the shared constant/type to whichever file
      actually consumes it (the real fix applied to `SMTP_POOL_SIZE_ENV`
      above), rather than leaving the cycle intact and relying on today's
      entrypoint import order. A later change to which file is the real
      entrypoint can reintroduce the exact same crash with zero changes
      inside the cycle itself.

## Checklist before adding a cross-package import

- [ ] Are both packages placed on the tier list above? Does the import point
      strictly downward? Same-tier or upward needs a registry/interface/event
      instead, never the direct import "just this once."
- [ ] If it's a new registry host, does it stay a passive store-and-return,
      never reacting to what a higher tier registered into it?
- [ ] If the dependency is downward, is it solving a real "this tier exists to
      provide that capability" need (persistence, kernel primitives) — not an
      avoidable abstraction some other package could host instead?
- [ ] Has the tier list itself been re-verified against real `deno.json`/import
      grep output recently, rather than trusted from memory of an earlier
      session?
- [ ] Registering a descriptor into an existing open registry (e.g. a new
      `TriggerActionJobsContainer` action kind)? Confirm this package
      actually owns the capability doing the real work before
      self-registering a working handler — if the work needs something this
      package doesn't already depend on, the registration belongs in
      whichever package does, mechanism-only here (see
      `datamaster-triggers`'s and `notifications-provider`'s own "registering
      a new trigger action" sections for both sides of this same check). The
      registry itself won't stop a second, unrelated package from also
      registering the same action kind for a different reason — it only
      throws on an exact duplicate key — so ownership is a design
      discipline, not something the container enforces for you.

## Out of scope — do not do these

- Designing a new mechanism *inside* `@zanix/server` itself (composition/
  activation layering, the core-slot registry, registration-function shape)
  — that's `zanix-server-internals`'s job; this skill only cares that
  `server` stays tier 0 from the outside.
- Deciding where an HTTP controller over a package's own data belongs (local
  admin API vs. a higher-tier aggregator proxying it) — that's
  `zanix-local-api-vs-aggregator`'s job. It's one consequence of the
  direction rule here, not a restatement of it.
- Shaping a new env var or selector inside a package that happens to be
  fixing a dependency-direction violation (e.g. the `TEMPLATES_BACKEND`
  rework referenced above) — that's `zanix-envvar-conventions`'s job; this
  skill only cares that the resulting adapter isolation holds.
- Deciding whether a cross-package coupling is documented or tested — that's
  `feature-completeness-conventions`/`docs-readme-audit`'s gate, not this
  one.
- Actually running a whole-monorepo violation sweep — that's a
  `dependency-direction-sweep` run's job (see the real `notifications ->
  auth` case above, found that way); this skill supplies the rules such a
  sweep checks against, not the sweep mechanism itself. Same division for
  the intra-package axis above: this skill states the rule and the manual
  checklist, `dependency-direction-sweep`'s own intra-repo check is the
  automated mechanism.
- Judging remediation priority/order across multiple violations a sweep
  turns up — report which tier each side sits at and which pattern (valid
  direct dependency, registry-drain, or confinement) applies; sequencing
  fixes across a whole ecosystem is a human call.

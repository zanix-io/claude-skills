---
name: datamaster-dlq
description: @zanix/datamaster's Mongo-backed Dead Letter Queue — lifecycle, lease-based concurrency, payload protection, and why it deliberately doesn't use the open-registry pattern datamaster-triggers uses. Use when registering the DLQ model, claiming/processing dead-lettered entries, or reviewing DLQ payload protection.
---

Distinct from `@zanix/asyncmq`'s own RabbitMQ-native DLQ mechanism — this is a
broker-agnostic, Mongo-backed dead-letter store any producer can push failed
work into, independent of which broker (if any) originated it. Distributed
processing of entries is handled by `@zanix/asyncmq/dlq`'s
`registerDLQProcessor`, not by this package directly. File:line references
point at `~/Documents/Development/ZanixLibraries/datamaster` — read the real
code there before assuming this summary is still accurate.

## Golden rule (token savings)

- The lifecycle state table and the two protection modes below are the fast
  path for reviewing DLQ usage — check against them directly rather than
  re-reading `DlqProvider`'s full source for a routine claim/complete/fail
  call.
- Confirm `payloadFields` vs whole-payload `encryptPayload` only when the
  schema shape actually matters (adding a new `processType`) — for ordinary
  claim/release/complete/fail calls, neither choice changes the call shape.

## Registration and access

```ts
import { registerDlqModel } from 'jsr:@zanix/datamaster@[version]'
registerDlqModel() // once per app, not once per processor
```

Resolved via `this.providers.get(DlqProvider)`, registered under the `'dlq'`
core-provider slot — no static worker slots needed beyond that.

**Older code/tests may still show `DLQProvider`/`registerDLQModel` (etc.,
all-caps `DLQ`)** — this package converged its DLQ symbols on `Dlq` casing
per `naming-and-structure-conventions`; the old names are kept as
`@deprecated` re-exports of the exact same binding (`mod.ts`'s "Deprecated
aliases" block), not a separate copy — safe to treat as identical, but write
new code against the `Dlq`-cased names above.

## Lifecycle

```
pending → claimed → completed
claimed → pending   (retry)
claimed → failed    (maxAttempts exceeded)
failed  → pending   (requeue())
```

## Concurrency

`claim()` is an atomic `findOneAndUpdate` — no external locking needed to
avoid two workers claiming the same entry. `release`/`complete`/`fail` all
require a matching `leaseOwner`, and throw `HttpError('CONFLICT')` if the
caller doesn't hold the current lease.

**Known limitation, worth checking before relying on lease exclusivity for
correctness**: a late `complete()` call from a lease holder whose lease has
already expired can still succeed — the lease-expiry window and the
`complete()` call's own validity aren't atomically joined. Don't assume an
expired lease is a hard guarantee against a stale worker's completion still
landing.

## Payload protection — two modes, and a real difference in failure behavior

- **`encryptPayload: true` / `{ type: 'asymmetric' }`** — all-or-nothing,
  changes the payload's storage shape from `Mixed` to an encrypted string,
  **loses queryability** entirely (you can't filter/project into an encrypted
  blob).
- **`payloadFields`** — field-level, takes priority over `encryptPayload` if
  both are set, compiles to **one schema shared by every `processType`** —
  namespace by nesting each `processType`'s own fields under its own key to
  avoid collisions between unrelated processors' payload shapes.

**Caution, don't transcribe this as a neutral fact**: DLQ payload encryption
**fails open** — a missing key env var stores the payload **unencrypted,
silently**, with no error and no loud warning at write time. This is the
opposite failure mode of `datamaster-storage`'s object encryption, which
**throws** on a missing/invalid key instead — the docs contrast this
explicitly, not as an oversight but as a real design difference. Don't assume
"payload protection is configured" means "payloads are actually being
encrypted" without checking the env var is genuinely present in the running
deployment — the two packages behave oppositely on this exact failure mode,
and it's easy to mentally carry one package's guarantee into the other.

## Why DLQ deliberately does NOT use the open-registry pattern `datamaster-triggers` uses

Hosting an open registry in `@zanix/datamaster` for DLQ distributed processing
(mirroring `TriggerActionJobsContainer`, drained by `@zanix/core`) looks like
the natural move by analogy with triggers — don't build it. That pattern
exists specifically to solve a *lateral* dependency problem (a package that
must never import a same-tier sibling directly). A DLQ processor doesn't have
that problem: `@zanix/asyncmq`
depending downward on `@zanix/datamaster` for `DlqProvider` is a perfectly
valid direct dependency (see `zanix-dependency-direction`), the same
direction `@zanix/notifications` already uses for its own persistence. Adding
an inversion layer where a plain direct import is already correct would be
unnecessary indirection, not a safer design — don't reach for
`registerTriggerActionJob`'s registry shape here just because it worked well
for triggers; the two problems aren't actually analogous.

## Env-var precedence — the one place this package inverts the general rule

Elsewhere in `@zanix/datamaster`, an explicit option always wins over its
matching env var (see `datamaster-database-and-models`). **DLQ inverts
this**: its env vars (`DLQ_MODEL_NAME`, `DLQ_ENCRYPT_PAYLOAD`,
`DLQ_DEFAULT_LEASE_MS`), when set, **always win over their matching
`registerDlqModel` option** — the opposite precedence, and the package's own
docs call this out explicitly as a deliberate exception rather than leaving
it implicit. Don't assume the "explicit option wins" rule generalizes here;
verify per-package (and per-module, in this one case) before relying on it.

## Distributed processing (handoff to `@zanix/asyncmq/dlq`)

```ts
// in @zanix/asyncmq, not this package
import { registerDLQProcessor } from '@zanix/asyncmq/dlq'
```

This package only provides the storage/lifecycle primitives (`DlqProvider`,
`claim`/`release`/`complete`/`fail`/`requeue`); actually running a scheduled
job that drains entries is `@zanix/asyncmq/dlq`'s responsibility, imported
from a **separate subpath** so pulling in the rest of `@zanix/asyncmq` never
drags in `@zanix/datamaster`'s module graph for apps that don't use this
feature.

## Local admin API (`@zanix/datamaster/dlq-api`) — which methods belong on it

`DlqAdminService`/`createDlqAdminController` (`dlq/dlq-api/`, mirroring
`triggers-api`'s own shape) expose only `push`/`get`/`list`/`requeue`/
`discard`/`remove` — genuine admin/operator actions. `claim`/`release`/
`complete`/`fail` are deliberately absent: they're lease-fenced by
`leaseOwner`, built for `@zanix/asyncmq/dlq`'s `registerDLQProcessor` (or
any other automated worker) to drive programmatically — an admin panel has
no real lease to present, so exposing them would mean either faking a
`leaseOwner` (risking collision with, or forcibly interrupting, a genuine
in-flight worker's own claim) or bypassing the fencing entirely. See
`dlq.service.ts`'s own JSDoc for the full reasoning — the worked example to
copy if this package ever grows a second admin-facing surface over a
lease-fenced resource.

## Checklist before adding a new DLQ-processed entity

- [ ] Is `registerDlqModel()` called exactly once for the whole app, not once
      per `processType`?
- [ ] If payload protection matters here, is the env var that backs it
      actually confirmed present in the deployment — not just assumed,
      given this fails open silently?
- [ ] If using `payloadFields`, is the new `processType`'s shape nested under
      its own key, so it can't collide with another processor's fields in
      the shared schema?
- [ ] Does anything here assume an expired lease is a hard exclusivity
      guarantee? It isn't — a late `complete()` can still land.
- [ ] Is `DLQ_MODEL_NAME`/`DLQ_ENCRYPT_PAYLOAD`/`DLQ_DEFAULT_LEASE_MS` set as
      an env var anywhere in this deployment — remember it silently overrides
      whatever `registerDlqModel` option was passed in code, the opposite of
      the rest of the package.

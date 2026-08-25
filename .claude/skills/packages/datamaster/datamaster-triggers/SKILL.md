---
name: datamaster-triggers
description: Declarative reactive triggers (mail/request/log/custom actions) on a model's create/update/delete lifecycle in @zanix/datamaster — the DSL, dispatch mechanism, interpolation, persisted/online-editable triggers, and the local /admin/triggers API. Use when adding a trigger to a model, registering a new trigger action type, or reviewing a trigger definition for correctness or security.
---

For where `extensions.triggers` attaches inside `registerModel`, see
`datamaster-database-and-models`. For the local-API-vs-aggregator pattern this
package's `/admin/triggers` controller follows, see the ecosystem-wide
`zanix-local-api-vs-aggregator` skill — this skill covers the trigger DSL and
dispatch mechanics themselves, not the general architectural rule. File:line
references point at `~/Documents/Development/ZanixLibraries/datamaster` —
read the real code there before assuming this summary is still accurate.

## Golden rule (token savings)

- The action-type table and the interpolation rules below are the fast path
  for writing a new trigger — apply them directly rather than re-deriving
  dispatch behavior by reading the trigger-dispatch source each time.
- Only read `TriggerActionJobsContainer`'s real source when actually
  registering a new action type (rare) — for using the four existing types
  (`mail`/`request`/`log`/`custom`), this skill's tables are enough.

## Security: never hardcode secrets in a trigger definition

A trigger's fields are declarative config that can be read back (the
persisted triggers collection is a plain document in the database) — a
literal API key, bearer token, or password written directly into `headers`,
`body`, `url`, or any other field is exposed to anyone who can read that
config, not just whoever executes the trigger.

```ts
// Don't:
headers: { authorization: 'Bearer sk_live_xxxxx' }
password: 'my-secret-password'

// Do — reference an env var instead:
headers: { authorization: 'Bearer ${{API_KEY}}' }
```

`${{ENV_VAR}}` resolves from `Deno.env` right before the action executes —
**as long as that variable is registered in the environment of the
application where the trigger actually runs**, not just wherever it was
authored.

## Declaring triggers (`extensions.triggers`)

```ts
registerModel({
  name: 'users',
  definition: { email: String, active: Boolean },
  extensions: {
    triggers: {
      post: {
        created: [{ mail: { to: '{{email}}', subject: 'Welcome {{name}}', body: { template: 'welcome', data: { name: '{{name}}' } } } }],
        updated: [{ request: { url: '${{USER_UPDATED_WEBHOOK_URL}}', method: 'POST', headers: { authorization: 'Bearer ${{WEBHOOK_TOKEN}}' }, body: { email: '{{email}}' } } }],
      },
      pre: { deleted: [{ custom: { name: 'archive-before-delete' } }] },
    },
  },
})
```

`Triggers` is `{ pre?, post? } × { created?, updated?, deleted? } →
Array<Partial<TriggerActions>>` — each event lists several actions, and one
action entry can mix `mail`, `request`, and `custom` together (all present
ones fire).

**`conditions`/`priority`/`delay`/`data` are independent per action type,
never shared** — even grouped on the same entry, each action type reads its
own copy. A `custom` action with `conditions` next to a `mail` action with
none means `custom` is conditional and `mail` fires unconditionally, every
time. To make `mail` conditional too, repeat `conditions` on `mail` itself.

## Trigger actions

| Action | Fields | Dispatched via |
| --- | --- | --- |
| `mail` | `to`, `subject`, + whatever the registered job expects | `DEFAULT_TRIGGER_JOBS.mail`, or a `registerTriggerActionJob('mail', ...)` override |
| `request` | `url`, `method`, `headers`, `body?` | `DEFAULT_TRIGGER_JOBS.request`, or a `registerTriggerActionJob('request', ...)` override |
| `log` | `level`, `message` | `DEFAULT_TRIGGER_JOBS.log` — self-registered by `datamaster` itself (`log-trigger.core.ts`), no consumer wiring needed |
| `custom` | `name` | The job named `name` — one already registered via `@zanix/asyncmq`'s `registerJob` |

Datamaster only knows `mail` needs a recipient and subject — the rest of the
payload is interpreted entirely by whichever job handles it. The default
`mail` job (self-registered by `@zanix/notifications` when bootstrapped via
`@zanix/core`) expects `body: { template: string; data?: Record<string,
unknown> | string }` — see `@zanix/notifications`'s own docs
(`MailTriggerActionData`, `sendMailTriggerNotification`) for that contract.

**`mail`/`request` need a consumer-side job registered to do anything.**
Dispatch goes through `ProgramModule.providers.get('worker')`, to whichever
job last registered via `registerTriggerActionJob`, falling back to
`DEFAULT_TRIGGER_JOBS`'s literal names. Apps bootstrapped via `@zanix/core`
get both wired automatically: `request` is registered by `@zanix/core` itself
(generic `fetch`); `mail` is self-registered by `@zanix/notifications`'s own
`/core` entrypoint. `@zanix/core` drains every descriptor registered this way
and performs the actual `@zanix/asyncmq` `registerJob` call — the one place
that happens, so a package registering its own trigger-action job never needs
to depend on `@zanix/asyncmq` directly (see `zanix-dependency-direction` for
the general pattern this is one instance of). Dispatch itself picks `runJob`
(queue-backed) when `AMQP_URI` is configured, or falls back to `runTask`
(local, in-process) otherwise.

### Registering a new trigger action type

`registerTriggerActionJob(actionKind, descriptor)` against
`TriggerActionJobsContainer` — an open registry with a specific, documented
rationale for existing (see `zanix-dependency-direction`'s worked example on
this exact mechanism). **This is the strongest repeatable "add a new X"
workflow in this package** — the two real precedents (`notifications`
registering `mail`, `@zanix/core` registering `request`) are the templates to
follow for a third.

**A new action type's real handler belongs in `datamaster` itself only if
the capability is already `datamaster`'s own dependency — otherwise it
belongs wherever that capability actually lives, and this package never
registers a working handler for it.** `TriggerActionJobsContainer`'s
descriptors are populated by whichever package owns the underlying
capability. `mail` (owned by `notifications`, self-registered from its own
`/core`) and `request` (generic outbound HTTP, owned by `@zanix/core`) both
cross into a sibling package's domain, so their real handlers live there —
`datamaster` only builds the shape (fields, dispatch entry in its own
typings/defs) for an action like that, and flags the missing handler
explicitly, never a working one, even when a task's phrasing reads like it
wants end-to-end behavior. `log` is the real counter-example: it writes via
`@zanix/logger`, which `datamaster` already imports as a real dependency
(check `deno.json(c)`'s `imports` and real usage — `@zanix/logger` shows up
across ~20 of this package's own files) — no package boundary is crossed,
so its handler is built and self-registered directly inside `datamaster`,
the same shape as `mail`'s own
`trigger-mail.ts`/`trigger-mail.core.ts` split in `notifications`, just
local. **The check before deciding**: does this action's real work require
a capability `datamaster` doesn't already depend on? If yes, mechanism-only
and flag the gap (`mail`/`request`'s shape). If no — the capability is
already a real `datamaster` dependency — build the working
handler here (`log`'s shape).

## Interpolation

Every string field on `mail`/`request`/`log` supports `{{field}}`/`{{nested.path}}`
placeholders, resolved against the record the trigger fired for — the
**only** way a field sees per-record data. This is `@zanix/utils`'s own
`interpolate` (real-typed-value-for-a-single-placeholder,
string-coerced-when-mixed — see `utils-interpolation-and-data-transforms`
for the general mechanic); applied here, a field like `amount:
'{{amount}}'` resolves to the record's real `amount` value (whatever
type), while `'Bearer {{apiKey}}'` always substitutes as a string.

For `GET`/`HEAD`/`DELETE`, a configured `request.body` is **converted into
query parameters** instead of a fetch body (many servers reject a body on
these methods): arrays expand into repeated keys (`tags=a&tags=b`), nested
objects use bracket notation (`address[city]=Bogotá`) — the same convention
`@zanix/utils`'s `getProcessedParams`/`toSearchParams` parses. For every
other method, `body` is sent as a real, JSON-serialized fetch body.

### `${{ENV_VAR}}` — independent second interpolation system

Resolved from `Deno.env`, fully independent of `{{field}}` — both can
coexist in the same string/object without interfering. Model interpolation
(`{{field}}`) always runs first, then `${{ENV_VAR}}` — this is
`@zanix/utils`'s own `interpolateEnv`; see `utils-interpolation-and-data-transforms`
for the real missing-var footgun (a missing variable silently becomes the
literal text `'undefined'`, never throws) — the same rule applies here
unchanged: check the real dispatched value, or that the env var is
genuinely set where the trigger actually runs, before trusting a trigger
that references one.

## Conditions

```ts
{ field: 'status', op: '=' /* '<'|'>'|'='|'<='|'>='|'includes'|'!=' */, value: 'active' }
```

An action's `conditions` array is evaluated with implicit AND. Nestable
logical groups: `{ or: [...] }`, `{ and: [...] }`, `{ not: [...] }`. `value`
supports two special forms beyond a literal: `'!$undefined'` (field was never
set), and a string starting with `$` to compare against another field on the
same data (`{ field: 'startDate', op: '<', value: '$endDate' }`).

## Document- and query-level coverage

Triggers hook **both** the document level (`.save()`) and the query level
(`updateOne`/`findOneAndUpdate`/`deleteOne`/`findOneAndDelete`) —
symmetrically, `pre` in the corresponding pre-hook, `post` in the
corresponding post-hook. **Not covered**: `insertMany` (bypasses document
middleware, isn't a single-document query either) and bulk operations
(`bulkWrite`, `updateMany`, `deleteMany`) — a trigger silently does not fire
for these, worth checking explicitly if a model's writes ever go through one.

## Dispatched payload

```ts
const args = {
  // ...the action's own fields, already interpolated against both {{field}} and ${{ENV_VAR}}
  priority, delay,
  data: {
    _data: {/* current document fields (or the deleted record, for a deleted trigger) */},
    _oldData: {/* pre-change document — only present for updated/deleted */},
    ...actionData, // the action's own `data` option, if set
  },
}
```

`args.data` is always present, for every action type including `custom` — a
custom job gets full, uninterpolated record access regardless of what
`mail`/`request` used via placeholders. `_timeout` (from
`TriggerActionCommons`) only applies on the local dispatch path (`runTask`,
when `AMQP_URI` isn't set), default `20_000`ms — silently dropped on the
queue-backed path (`runJob`), which has no timeout counterpart to forward it
to.

**Data protection is reversed before dispatch** — every document a trigger
sees (current record, `_oldData`, deleted record) has protected paths already
decrypted/unmasked (hashed paths dropped, same as a normal client-facing
read). A trigger's `{{field}}` interpolation and `conditions` never see raw
ciphertext or a hash value.

## Persisted triggers (online adaptation)

Triggers can be added/edited/toggled at runtime via the internal collection
(`ZanixMongoConnector`'s `triggersModel` option, default `'zanix-triggers'`,
`false` to disable). Two kinds of entry:

- **Auto-seeded** (`isDefault: true`) — every model with a static
  `extensions.triggers` gets a matching entry auto-created on first boot.
  From then on, **this entry is the sole source of truth**; the static
  config never fires directly again. Deleting it doesn't stick (re-seeded
  fresh on next boot from code) — **`active: false` is the only way to
  durably turn a code-defined trigger off**. An auto-seeded entry stays in
  sync with code on every boot, EXCEPT once it's been edited directly in the
  database — from that point, "a manual edit always wins over a later code
  change," permanently, until the row is deleted (which resets it to
  whatever code currently says).
- **Created from scratch** (`isDefault: false`/absent, e.g. via an admin
  endpoint) — **combines with**, never replaces, the model's static
  `extensions.triggers`. Both sets of actions run.

`mail`/`request`/`log` work fully from the database alone (well-known job
names, no code change needed — `log`'s handler ships self-registered);
**`custom` does not** — unless the job name was
already registered via `@zanix/asyncmq`'s `registerJob` somewhere in code, a
database-only `custom` entry has nothing to run.

**Two caveats, easy to miss**:
- Auto-seeding only sees models registered before `connect()` (standard
  `registerModel` at module-load time) — a model bound dynamically via
  `getModel(name, schema, {extensions})` *after* the connector already
  connected won't be auto-seeded that boot; its static trigger fires
  directly and normally instead, just not yet editable from the database.
- `triggersModel: false` means "only code triggers" for *that connector* —
  on startup a connector always resets its own in-memory persisted-triggers
  state first, so it never inherits state from a prior instance of itself.
  Scoped per connector, same as models/seeders.

### Keeping the registry fresh without a restart

Three layered mechanisms, each covering what the others can't:

| Mechanism | Enabled by | Speed | Covers |
| --- | --- | --- | --- |
| On-write refresh | Always on | Instant | This connector's own model only |
| Polling | `triggersPollInterval` (ms) / `TRIGGERS_POLL_INTERVAL` | Up to one interval | Any process/replica, or a direct DB edit |
| Change Stream | `triggersChangeStream: true` / `TRIGGERS_CHANGE_STREAM` | Near-instant | Any process/replica — requires a replica set/sharded cluster |

For a horizontally-scaled deployment, on-write refresh only updates the
replica that made the write — polling or Change Streams propagate to other
replicas. On a replica set, Change Streams alone gives near-instant sync
everywhere; off a replica set, polling is the only cross-replica option.
Change Stream against a standalone instance fails to start — logged, not
fatal; the connector keeps running on the other two mechanisms.

## Local `/admin/triggers` API

```ts
import { createTriggersAdminController } from 'jsr:@zanix/datamaster@[version]/triggers-api'

const TriggersAdminController = createTriggersAdminController({
  guards: [jwtValidationGuard], // omit for no auth at all — this package assumes none by default
  versionProtocol: { version: 1 },
})
```

This package owns both the data and the local HTTP surface fronting it — the
same "local API lives with its domain" shape `@zanix/space`'s `assets-api`
establishes (see `zanix-local-api-vs-aggregator`). `@zanix/admin`'s own
triggers concern (`TriggersAggregator`) is a genuinely different one —
cross-service aggregation, proxying over N services' own local
`/admin/triggers` APIs. Route prefix (`admin/triggers`) is fixed, not
configurable — it's the wire-protocol contract `TriggersAdminClient`
hardcodes. `TriggersAdminRepository`/`TriggersAdminService` are exported as
ready-made CRUD data access/business logic over the persisted-triggers
collection; `createTriggersDiscoveryProvider()` builds the
`/.well-known/zanix/triggers` Discovery provider `@zanix/admin` composes.

## Checklist before adding or reviewing a trigger

- [ ] Any secret referenced via `${{ENV_VAR}}`, never hardcoded in the
      trigger definition?
- [ ] Does the target env var actually resolve where the trigger runs — not
      just where it was authored? A missing one silently becomes the text
      `'undefined'`, not an error.
- [ ] Does `mail`/`request` have a real job registered to handle it (default
      or overridden via `registerTriggerActionJob`)? (`log` always does —
      self-registered by `datamaster` itself.) Does `custom` reference a job
      name that's actually registered via `@zanix/asyncmq`?
- [ ] `conditions`/`priority`/`delay`/`data` set on the action(s) that
      actually need them — not assumed shared across a multi-action entry?
- [ ] Does this trigger need to fire on `insertMany`/bulk operations? If so,
      it won't — that path needs separate handling.
- [ ] For a persisted/database-editable trigger: is the distinction between
      auto-seeded (replaces static) and scratch-created (combines with
      static) understood for this specific case?

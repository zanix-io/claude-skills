---
name: datamaster-database-and-models
description: ZanixMongoConnector, the registerModel DSL, seeders, multi-database and multi-connector support, the SQLite KV store, and pagination/search statics — the core of @zanix/datamaster. Use when defining a new model, adding a second Mongo connector, or reviewing how a model/seeder/connector is wired.
---

For how a new connector *slot itself* gets registered (the mechanism `getModel`
and `registerModel` build on), see `datamaster-connector-registration`. For
what attaches to a model via `extensions` beyond seeders (triggers, data
protection), see `datamaster-triggers` and `datamaster-data-protection`.
File:line references point at `~/Documents/Development/ZanixLibraries/datamaster`
— read the real code there before assuming this summary is still accurate.

## Golden rule (token savings)

- Confirm a specific `registerModel`/`getModel` option's real shape with a
  targeted grep/type lookup, not a full read of the connector's source —
  this skill's own tables already cover the common options; verify only what
  you're actually about to use differently from here.
- When adding a new model, copy the shape from an existing sibling model in
  the target repo (or this skill's own examples) rather than re-deriving the
  `definition`/`extensions`/`callback` structure from scratch each time.

## `ZanixMongoConnector`

```ts
const connector = new ZanixMongoConnector({
  uri: Deno.env.get('MONGO_URI'), // falls back to MONGO_URI, then 'mongodb://localhost'
  seedModel: 'my-seed-register-model', // default: 'zanix-seeders'; false disables seed tracking
  triggersModel: 'my-triggers', // default: 'zanix-triggers'; false disables persisted triggers
  triggersPollInterval: 5000, // default: false (disabled)
  triggersChangeStream: true, // default: false; requires a replica set/sharded cluster
  config: { dbName: 'my_database' },
})
await connector.isReady // the property to await — there's no `connectorReady`
```

Every option has an env-var counterpart (`uri`→`MONGO_URI`, `seedModel`→
`SEED_MODEL_NAME`, `triggersModel`→`TRIGGERS_MODEL_NAME`,
`triggersPollInterval`→`TRIGGERS_POLL_INTERVAL`, `triggersChangeStream`→
`TRIGGERS_CHANGE_STREAM`) — **the explicit option always wins if both are
set**, the general rule this package follows almost everywhere (DLQ is the
one documented exception — see `datamaster-dlq`).

### `getModel` — three overloads

```ts
// 1. From a schema you provide directly
const Model = connector.getModel('users', schema, { useALS: true })

// 2. From a plain definition — connector builds the Schema for you
const Model = connector.getModel<Attrs>('users', {
  definition: { name: { type: String, required: true } },
  extensions: { triggers: { post: { created: [{ custom: { name: 'my-job' } }] } } },
})

// 3. Look up an already-registerModel-bound model (throws if not found)
const Model = connector.getModel<Attrs>('users')
```

`useALS: boolean` re-enters the request's `AsyncLocalStorage` session context
before resolving — enable it when the calling handler already has
`useALS`/`enableALS` active, so accessors like `dataAccessGetter` (see
`datamaster-data-protection`) see the session. `onSeedersDone?: (Model, msg) =>
void` is the schema-overload's way to wait for `extensions.seeders` to finish,
since `getModel` itself returns synchronously before seeders settle.

## `registerModel` DSL

```ts
registerModel<Attrs>({
  name: 'users',
  definition: {
    name: String,
    email: { type: String, get: dataPoliciesGetter({/* ... */}) },
  },
  extensions: {
    seeders: [async function seedAdmin(Model) {
      const exists = await Model.findById('...')
      if (exists) return
      return new Model({ id: '...', name: 'Admin' }).save()
    }],
    // triggers: ... — see datamaster-triggers
    // autoProtectOnUpdate: true — see datamaster-data-protection
  },
  callback: (schema) => {
    schema.index({ name: 1 })
    return schema // must return it, possibly modified
  },
})
```

`extensions.seeders` mixes two forms: a plain `(Model, connector) => void |
Promise<void>` function, or `{ handler, options: { version, verbose,
runningMode } }` for more control. Seeders run **sequentially**, never in
parallel — a later seeder can rely on an earlier one having already committed.

### Seeders shortcuts: `seedByIdIfMissing`/`seedManyByIdIfMissing`

```ts
extensions: {
  seeders: [
    seedByIdIfMissing({ id: '...', name: 'Admin' }),
    seedManyByIdIfMissing([{ id: '...', name: 'A' }, { id: '...', name: 'B' }]),
  ],
}
```

Both **upsert by `id`** — safe to re-run, existing documents are left
unchanged. Both default `useDataPolicies: true`, so protected fields go
through mask/encrypt/hash on the way in exactly like a normal save — pass raw
plaintext, let the schema's setter protect it. **Pass `{ useDataPolicies:
false }` only when the seed data is already protected** (a literal export
from production, with the version prefix already baked in like `'v0:...'`) —
otherwise the setter double-protects an already-protected value. Skip all
seeders for a run via the `DATABASE_SEEDERS` env var.

## Multi-database support (`'database:model'` prefix)

```ts
registerModel({
  name: 'billing:invoices',
  definition: { customer: { type: Schema.Types.ObjectId, ref: 'accounts:users' } },
})
const Invoices = connector.getModel('billing:invoices')
```

Use the full prefixed string consistently, both in `getModel` and in any
`ref` targeting it. **The package's own docs flag this explicitly as "not
recommended for microservices"** — prefer one independent database per
service; this convention exists for monoliths/shared-database scenarios where
splitting isn't an option, not as a general-purpose pattern to reach for.

## Multiple Mongo connectors

`registerModel` targets the **default** connector (the `'database'` core
slot) unless told otherwise — a model for a second connector must say so
explicitly, or the default connector binds it instead:

```ts
@Connector({ slot: 'billing' })
class BillingConnector extends ZanixMongoConnector {
  constructor() { super({ uri: 'mongodb://billing-host/billing' }) }
}

registerModel({ name: 'orders', definition: { total: Number } }) // → default connector
registerModel({ name: 'invoices', definition: { amount: Number } }, BillingConnector) // → BillingConnector
```

Pass the connector **class**, not a slot string — `registerModel` resolves
its identity the same way `@zanix/server`'s DI container does internally, so
it works whether or not `BillingConnector` has an explicit `slot`. The class
must already be `@Connector`-decorated (imported) by the time `registerModel`
runs, or the call **throws immediately, naming the connector** — a
deliberate fail-fast, since a silent mismatch here would otherwise surface
later as a confusing "model not found" from a completely different
connector's `getModel()`.

`getModel()` for a model registered under a *different* connector throws
`ERR_MONGO_MODEL_NOT_FOUND`, naming which connector(s) it IS registered for —
`error.meta.kind` is `'wrong-connector'` vs `'never-registered'` for
programmatic handling, distinct failure modes worth branching on if you
handle this error explicitly.

**Seeders and persisted triggers follow the same per-connector isolation as
models** — each connector only reads/writes its own bucket, so two genuinely
different connectors never share or clobber state, even pointed at the same
physical database.

## SQLite key-value store

```ts
class MyKVStore extends ZanixKVStoreConnector<string> {}
const kv = new MyKVStore({ filename: 'my-store.sqlite' }) // default: 'znx.kv.tmp'
await kv.set('key', 'value', 60) // TTL in seconds; 'KEEPTTL' preserves current expiration
const value = await kv.get('key')
```

**TTL expiry is lazy** — an expired entry is skipped on read, never
proactively deleted; don't assume disk/memory is reclaimed the moment a TTL
elapses. `withLock` uses the same internal keyed lock manager as
`datamaster-cache`'s `withLock` — see that skill for its single-process,
non-distributed caveat, which applies identically here. For direct SQLite
table access without KV/TTL semantics, use `LocalSQLite(table, filename?)`.

## Pagination and search

```ts
const page = await UsersModel.paginate({ page: 1, limit: 10, filter: {}, sort: { _id: 1 } })
// { docs, page, limit, total, totalPages, hasNextPage, hasPrevPage }

const cursorPage = await UsersModel.paginateCursor({ limit: 10, cursor: page.docs.at(-1)?._id })
// { docs, limit, nextCursor, hasNextPage }
```

Both accept `useDataPolicies: true` — protects `filter`'s `mask`-strategy
paths before the query runs (see `datamaster-data-protection`).

`search: { query, fields }` builds a partial-match `$or` across `fields`,
**combined with `filter` as `{ $and: [search, filter] }`, never merged into
it** — an `$or`/`$and` already present in `filter` is never silently
overwritten by the search's own `$or`. Each field is checked against its own
protection config: unprotected → plain case-insensitive substring match;
`mask`-protected → search term masked first, matched as a **prefix** (masking
is deterministic and position-keyed, so only a term starting at index 0
guarantees a matching prefix); `hash`/`encrypt`-protected → **throws**, no
partial match is possible against what's actually stored. This is sugar over
`Model.buildSearchFilter(query, fields, extraFilter?)`, callable directly to
build/merge a filter yourself.

## Checklist before adding a new model

- [ ] Does `definition` use `dataProtectionGetter`/`dataAccessGetter` for any
      field that needs it (see `datamaster-data-protection`), rather than
      storing sensitive data in the open?
- [ ] Does `extensions.seeders` use `seedByIdIfMissing`/`seedManyByIdIfMissing`
      where it fits, rather than hand-rolling an upsert-by-id check?
- [ ] If this model belongs to a non-default connector, is the connector
      class passed as `registerModel`'s third argument — not left implicit
      (which silently binds it to the default connector instead)?
- [ ] Does `callback(schema)` actually `return schema`? Omitting the return
      silently drops any indexes/methods it added.
- [ ] Is `'database:model'` multi-database prefixing being reached for by
      default, or only because a real monolith/shared-database constraint
      requires it? The docs' own warning against it for microservices holds.

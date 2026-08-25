---
name: datamaster-connector-registration
description: How @zanix/datamaster registers a new core connector slot — the registerCoreConnectorSlot + @Connector-decorated class + env-var-gated auto-registration pattern, identically repeated across mongo/redis/qlru/memcached/sqlite/observability/storage. Use when adding a new connector backend to this package, or reviewing whether an existing one follows the convention.
---

This is the single most repeatable workflow in `@zanix/datamaster` — every one
of its 7 connector backends (Mongo, Redis, local-LRU cache, Memcached, SQLite,
Elasticsearch/ OpenSearch, S3-compatible storage) registers itself the identical
way, in each module's own `core.ts`. File:line references point at
`~/Documents/Development/ZanixLibraries/datamaster` — read the real code there
before assuming this summary is still accurate; it will drift as the package
evolves.

## Golden rule (token savings)

- **Read one `core.ts` as the template, not all six.** The pattern is identical
  across `database/providers/mongo/connector/mod.ts`,
  `cache/providers/redis/core.ts`, `cache/providers/qlru/core.ts`,
  `database/providers/sqlite/core.ts`, `observability/core.ts`,
  `storage/core.ts` — pick the one closest to what you're building (a new cache
  backend → read `redis/core.ts`; a new database backend → read
  `mongo/connector/mod.ts`) and adapt it, don't diff all six against each other
  first.
- Confirm the slot key/env-var gate with a targeted grep, not a full read of the
  connector's whole implementation — the registration shape is small and
  separate from the connector's actual domain logic.

## The pattern

Every connector's `core.ts` does two structurally separate things, and keeps
them separate on purpose:

1. **Unconditional slot registration** —
   `registerCoreConnectorSlot(key,
   BaseTarget, {sourcePackage})` runs at
   module-load time, always, regardless of whether the backend is actually
   configured for this deployment. This is what makes the slot exist in the
   registry at all (see `zanix-server-internals` for the mechanism itself,
   shared with `@zanix/server`'s own core-slot design).
2. **Conditional instance construction/registration** — the actual connector
   _instance_ only gets constructed/wired when the deployment is configured for
   it (an env var like `REDIS_URI`/`SEARCH_ENGINE` is set).
   A slot existing in the registry and a slot actually being backed by a live
   connector are two different things — don't conflate "registered" with
   "active."

Real slot keys, one per connector (verify against source before adding a new one
— this list will grow):

| Slot key            | Connector                                                 | Registered in                                  |
| ------------------- | --------------------------------------------------------- | ---------------------------------------------- |
| `'database'`        | `ZanixMongoConnector`                                     | `database/providers/mongo/connector/mod.ts:50` |
| `'cache:redis'`     | `ZanixRedisConnector`                                     | `cache/providers/redis/core.ts:33`             |
| `'cache:local'`     | `ZanixQLRUConnector`                                      | `cache/providers/qlru/core.ts:30`              |
| `'cache:memcached'` | `ZanixMemcachedConnector`                                 | `cache/providers/memcached/core.ts:34`         |
| `'kvLocal'`         | SQLite KV store                                           | `database/providers/sqlite/core.ts:30`         |
| `'search'`          | `ZanixElasticsearchConnector` OR `MeilisearchConnector`†  | `observability/core.ts:57`                     |
| `'s3'`              | `S3ObjectStorage` (renamed from `SeaweedFSObjectStorage`) | `storage/core.ts:40`                           |

† `'search'` is the one slot with two real, shipped, mutually-exclusive
implementations today, selected via `SEARCH_ENGINE=elasticsearch|opensearch|
meilisearch` (see the selector-pattern section below) — before assuming any
slot in this table maps to exactly one connector class going forward.

**`'s3'` isn't one of `@zanix/server`'s six hardcoded core slots** (`cache`,
`database`, `asyncmq`, `worker`, `kvLocal`, `search`) — so there's no dedicated
`this.s3` getter on `CoreBaseClass`. Resolve it via `this.connectors.get('s3')`,
the same way `'auth'`/`'notifications'` (which also have no dedicated getter)
are resolved — see `zanix-server-internals`'s getter-criterion section for why a
new slot doesn't automatically earn one.

**Adding a genuinely new cache backend goes under `'cache:custom'`, not a new
pre-seeded slot** — the same genericness call already made for `'s3'` below,
applied to cache. `CoreCacheConnectors` (`@zanix/server`) is a closed type:
`'local' | 'memcached' | 'redis' | 'custom'`, and `'custom'` is the
deliberate escape hatch for exactly this case (no dedicated `.custom` getter
either, same as `'memcached'` — resolve via `this.cache.use('custom')`).
Apply the same protocol-vs-config-change test used for `'s3'` (see below):
a backend that speaks Redis's own wire protocol (KeyDB, Dragonfly, ...) is a
config change under `'cache:redis'`, not a new class or slot; only a
genuinely different protocol needs its own `ZanixCacheConnector` subclass
under `'cache:custom'`. Extending `CoreCacheConnectors` itself with a new
named type (earning a dedicated getter like `.redis`/`.local`) is real
`@zanix/server` architecture work gated by `zanix-server-internals`'s
getter-criterion — not something to do unilaterally from this package just
because a new backend feels important enough to deserve one.

**The two parts of the pattern don't always live in the same file.** For Mongo
specifically, the unconditional `registerCoreConnectorSlot` call lives in
`mod.ts` (guaranteed just by importing `ZanixMongoConnector` — deliberately
_not_ gated behind `core.ts` or `MONGO_URI` ever running), while the conditional
connector registration (`registerMongoConnector`, gated on `MONGO_URI`) lives in
`core.ts`. Don't assume both halves are always in one `core.ts` file — confirm
per connector, the same way you'd confirm any other file:line claim in this
skill.

**Caution when splitting the pattern across files**: `registerCoreConnectorSlot`
running at module load is an eager side effect — exactly the idiom that caused
a real, shipped crash in `@zanix/notifications`'s SMTP connector, where the
same unconditional-top-level-call shape closed an accidental import cycle
across three files (`defs.ts`/`connector.ts`/`pool.ts`) and threw
`ReferenceError: Cannot access 'SmtpClient' before initialization` the first
time the connector actually activated — see `zanix-dependency-direction`'s
"intra-package circular imports with a top-level side effect" section for the
full precedent and the rule. The idiom itself isn't the bug — every connector
in this table uses it safely — but splitting `registerCoreConnectorSlot`'s
call site from the class/constant it reads across two-or-more files (as
Mongo's `mod.ts`/`core.ts` split already does) is exactly the shape that can
accidentally close a cycle if a future edit adds a back-import. Check for one
before shipping a new multi-file connector split, don't assume it's safe just
because Mongo's own split has stayed safe so far.

## Registration functions are exported, not just auto-run

Every connector's conditional registration function (`registerMongoConnector`,
`registerS3Connector` (renamed from `registerSeaweedFSConnector`),
`registerRedisConnector`, `registerQLRUConnector`, `registerKVConnector`,
`registerElasticsearchConnector`, `registerCacheProvider`,
`registerDLQProvider`) is exported from its owning module, reachable via
`@zanix/datamaster/core` — not just a private side effect run once at import
time. It still runs automatically on import, same as always; the export adds a
second way to invoke it: **re-registering after clearing the relevant registry**
(`closeAllConnections()`/`ProgramModule.targets.resetContainer(['type:connector'
|'type:provider'])`,
both from `@zanix/server`) without needing a fresh module evaluation — for a
config-reload in a long-running process, or a test simulating a different env
state between cases:

```ts
import { registerMongoConnector } from "@zanix/datamaster/core";

ProgramModule.targets.resetContainer(["type:connector"]);
Deno.env.set("MONGO_URI", "mongodb://new-host/db");
registerMongoConnector(); // re-applies the @Connector decorator against the new env state
```

The same pattern was adopted across `@zanix/auth`, `@zanix/asyncmq`,
`@zanix/notifications`, and `@zanix/app` in the same batch of work — if you're
extending one of those packages' own connector/provider registration and need
the same re-registration capability, this is the precedent to follow, not
something to design fresh.

## A single-instance slot that could get a second, competing implementation needs an explicit selector — not a conflict guard

`registerCoreConnectorSlot` makes a slot **exist**; it does nothing to stop two
different `@Connector`-decorated classes from both targeting the same slot.
Verified empirically for `'search'`: registering
`ZanixElasticsearchConnector` then `MeilisearchConnector` (or the reverse)
against the same slot leaves only the **second one registered — no error, no
warning**. If a slot could plausibly have more than one real backend gated by
different env vars (today: `'search'` — ES/OpenSearch vs. Meilisearch; `'s3'`
is currently safe since every S3-compatible backend is the same generic client
under a config change, not a second class), that's a real silent-misconfiguration
risk, not a hypothetical.

**Superseded (2026-08-20): the fix is a single `X_MODE=a|b|c` selector env var
+ one small resolver function, not a pairwise `assertXNotConflicting()` guard
watching two separate per-backend vars.** The guard shape below was this
package's own real, shipped design until the ecosystem-wide env-var audit
found it was the wrong direction to generalize — see `zanix-envvar-conventions`
for the full four-pattern taxonomy this decision is grounded in, and don't
reach for a new pairwise guard for a future mutually-exclusive slot; reach for
the selector shape instead.

`'search'`'s own real, current implementation: `SEARCH_ENGINE_ENV`/
`SEARCH_URL_ENV` + `resolveSearchEngine()` (`observability/search-config.ts`)
— `SEARCH_ENGINE=elasticsearch|opensearch|meilisearch` picks the backend
class, `SEARCH_URL` is read by whichever one gets selected. Replaced the old
per-backend `ELASTICSEARCH_URL`/`OPENSEARCH_URL`/`MEILISEARCH_URL` vars
entirely — with the selector, two backends configured for the same deployment
can no longer be REPRESENTED, let alone silently collide; there's nothing left
for a guard to catch after the fact. `@zanix/notifications`'s
`TEMPLATES_BACKEND_ENV` + `templatesBackendMode()`
(`modules/templates/provider.ts`) is the sibling precedent for the identical
shape, applied to `notifications`' own templates-persistence mode instead of a
datamaster core-connector slot — read either as the literal template for a
new mutually-exclusive slot in THIS package. Both replaced an older
`assertXNotConflicting()`-shaped guard (`assertSearchConfigNotConflicting()`/
`assertTemplatesConfigNotConflicting()`), now fully removed from both
packages — don't resurrect that pattern, and don't cite either function name
as if it still exists.

**A new mutually-exclusive slot in this package**: add `<SLOT>_ENGINE_ENV`
(or a name matching the slot's own domain) + a `resolve<Slot>Engine()`
function mirroring `resolveSearchEngine()`'s exact shape (`returns <Mode> |
undefined`, throws a real `InternalError` naming the invalid value and the
supported set if set to anything unrecognized) — call it from inside every
conditional registration function for that slot, gating each one on whether
the resolved mode is the one it owns. See `zanix-envvar-conventions`'s own
checklist before building this — it also covers the hard-rename migration
cost (grepping every OTHER Zanix repo for an import of whatever old
per-backend vars get removed) that both real fixes here had to account for.

## One remaining future/placeholder connector, verified empty — not stale docs

`database/providers/postgres/` exists as a real directory in source but contains
**zero files** — this genuinely means "documented as coming soon, not yet
implemented," not a doc that quietly fell behind a real implementation. Confirm
this with a directory listing before assuming it has any real code to reference
— it's an easy claim to get wrong by assuming a directory implies an
implementation. `cache/providers/memcached/` was this same kind of placeholder
until it shipped a real connector (`ZanixMemcachedConnector`,
`cache/providers/memcached/core.ts`) — a genuine precedent that these
placeholders do eventually get filled in, not permanent fixtures; re-verify with
a fresh directory listing before trusting this section at all, the same
discipline that caught memcached's own staleness.

## Every public command method must gate on `isReady` first

A connector's public methods (`set`/`get`/`delete`/etc.) must await
`this.isReady` (or a bounded wrap around it, like `ZanixMemcachedConnector`'s
own `#ready()`) before touching connection state — never assume `initialize()`
has already finished just because the happy path usually reaches it in time.
`ZanixRedisConnector`'s real safety net for this is `execWithRetry`
(`cache/providers/redis/connector/retries.ts`), which isn't obvious from
`redis/core.ts` alone — read the connector's actual command methods
(`connector/mod.ts`, not just `core.ts`) when copying this pattern, not only the
registration half. Skipping this surfaces as a `TypeError` on an unset field,
and only under real timing pressure (a slow or failed initial connection), not
on a happy-path smoke test — a genuinely easy gate to miss by templating only
the registration shape.

## Checklist before adding a new connector slot

- [ ] Does `registerCoreConnectorSlot` run unconditionally at module load
      (wherever it actually lives — `mod.ts` and `core.ts` both occur in
      practice) — never gated behind the same env-var check that gates instance
      construction?
- [ ] If the slot registration and the connector class/constant it reads are
      split across more than one file, run `zanix check-cycles --path .`
      (the real, built tool) to confirm those files don't form an import
      cycle. If one exists, does any file in it read a cross-cycle binding
      at its own top level — the exact shape that produced a real crash in
      `@zanix/notifications`'s SMTP connector (see the caution above and
      `zanix-dependency-direction`'s "intra-package" section)?
- [ ] Is the actual connector instance only constructed/wired when its
      deployment-specific env var is actually set — never eagerly, even if the
      slot itself is always registered?
- [ ] Does the new slot need a dedicated `CoreBaseClass` getter? Per
      `zanix-server-internals`'s criterion, almost certainly not — resolve via
      `this.connectors.get('<slot>')` like `'s3'`/`'search'` already do.
- [ ] Is the conditional registration function itself exported (reachable via
      `@zanix/datamaster/core`), not left as a private side effect — so a caller
      can re-register it after a registry reset, matching every existing
      connector's own convention?
- [ ] Is the slot key added to this skill's table once the connector ships, so
      the next person building one has a real, current reference list?
- [ ] If this slot could plausibly get a second, competing implementation (a
      new class targeting a slot that already has one), does a single
      `<SLOT>_ENGINE`-shaped selector env var + `resolve<Slot>Engine()`
      function exist — mirroring `resolveSearchEngine()`/
      `templatesBackendMode()`'s shape — rather than a pairwise
      `assertXNotConflicting()` guard (see `zanix-envvar-conventions`, and
      this skill's own section above)? If a selector already exists for this
      slot (a 2nd backend becoming a 3rd), does its resolved-value set
      actually include the new mode, not just the call sites re-copied?
- [ ] Does at least one test resolve the new connector through the layer a real
      caller actually reaches it by (`this.cache.use('<suffix>')`/
      `this.connectors.get('<slot>')`), not just exercise the connector class
      standalone or the `core.ts` DSL's own env-var gating? Those two prove the
      class works and the slot registers/skips correctly — neither one proves a
      consumer can actually retrieve and use the connector the way it's meant to
      be reached. `cache/providers/memcached/` shipped without this (caught,
      then added, only in review) despite full unit/integration/ DSL-gating
      coverage — `src/@tests/functional/cache/provider.test.ts`'s own
      `provider.redis` tests are the real precedent for what this looks like for
      a cache connector; the equivalent exists per-area for
      database/storage/observability slots too.
- [ ] If the new connector talks to a real network service (not a pure
      in-process implementation), does a **functional suite against a real
      instance** of that service exist — gated by its own `RUN_X_TESTS=true`
      flag, `ignore`d by default so a plain `deno test --allow-all` never
      needs Docker — not just a unit suite mocking `fetch`/the client SDK?
      Mocked unit tests only prove the connector sends what it *believes* the
      real protocol expects; they can't catch a real instance behaving
      differently than assumed. Confirmed concretely: `MeilisearchConnector`
      shipped unit-only for a real stretch of time before this suite
      (`src/@tests/functional/observability/meilisearch-connector-real.test.ts`,
      mirroring the ES/OpenSearch and S3/SeaweedFS functional suites already
      documented per-area) caught a real, non-obvious fact no mock would ever
      surface: Meilisearch fails a write task atomically (the WHOLE batch, not
      per-document like ES's `_bulk`) — see `datamaster-observability` for the
      specifics. Wire the new `RUN_X_TESTS` flag into `.env.test.example` AND
      `.github/workflows/publish.yml` (a `services:` block if the image's
      default entrypoint needs no extra args, a manual `docker run` step
      otherwise — see that workflow's own SeaweedFS-vs-OpenSearch contrast for
      which shape applies), not just left as a local-only opt-in.

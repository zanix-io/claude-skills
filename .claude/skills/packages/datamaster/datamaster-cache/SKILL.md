---
name: datamaster-cache
description: @zanix/datamaster's Redis, Memcached, and local-LRU cache connectors, and the multi-layer ZanixCacheCoreProvider (getCachedOrFetch/getCachedOrRevalidate/withLock). Use when caching a value, choosing between Redis/Memcached/local cache, or reviewing a cache-invalidation/locking pattern.
---

For how the `'cache:redis'`/`'cache:memcached'`/`'cache:local'` slots
register themselves, see `datamaster-connector-registration`. File:line
references point at `~/Documents/Development/ZanixLibraries/datamaster` —
read the real code there before assuming this summary is still accurate.

## Golden rule (token savings)

- The two connectors' option tables below are the fast path — confirm a
  specific default against source only when actually relying on a
  non-obvious one (e.g. `reconnectStrategy`'s formula), not for routine use.

## `ZanixRedisConnector`

```ts
new ZanixRedisConnector({
  redisUrl: Deno.env.get('REDIS_URI'), // → REDIS_URI
  ttl: 0, // 0 = never expires
  maxCommandRetries, commandTimeout, commandRetryInterval, connectionTimeout,
  reconnectStrategy, // default: min(retries * 100, 5000)ms
})
```

**Caution, a real footgun**: `clear()` flushes the **entire Redis database**,
not just keys this connector wrote. If the same Redis instance is shared with
anything else (another app, another connector, a different key namespace),
`clear()` wipes all of it — never call it against a shared instance without
confirming nothing else depends on that database.

Command scheduling: `{ schedule: true }` batches writes into a
`RedisPipelineScheduler`, flushed at `maxBatch` (default 200) or `maxDelay`
(default 100ms), whichever comes first.

## `ZanixMemcachedConnector`

```ts
new ZanixMemcachedConnector({
  memcachedUri: Deno.env.get('MEMCACHED_URI'), // → MEMCACHED_URI, default 'localhost:11211'
  ttl: 0, // seconds — whole-second granularity, no Redis-style sub-second PX
  connectionTimeout,
})
```

Classic Memcached text protocol over a raw `Deno.connect` TCP socket — no
external client dependency, unlike Redis's `redis` npm package.

**Caution, a real footgun**: `keys()`/`values()`/`size()` are backed by a
**per-instance, in-memory key index**, not the server — the classic protocol
has no `SCAN`/`DBSIZE` equivalent, and the alternative (`stats
cachedump`) is commonly disabled in production. A key written by a
*different* connector instance, process, or client talking to the same
server is invisible to these three methods; they're always only a **lower
bound** on what the server actually holds. `get`/`set`/`has`/`delete`/`clear`
all talk to the real server directly and don't have this limitation.

**Caution, a real footgun (shared with Redis)**: `clear()` flushes the
**entire Memcached instance** (`flush_all`) — same shared-instance risk as
`ZanixRedisConnector.clear()`.

**`exp: 'KEEPTTL'` throws** (`MEMCACHED_KEEPTTL_UNSUPPORTED`) instead of
silently guessing — the protocol exposes no command to read a key's
remaining TTL, so there's no way to genuinely preserve it on overwrite.

`ZanixCacheCoreProvider`'s multi-layer methods
(`getCachedOrFetch`/`getCachedOrRevalidate`/`saveToCaches`) stay
**Redis-only** — typed against `Extract<CoreCacheConnectors, 'redis'>`,
Memcached isn't wired into them. It's still reachable at the lower
`ZanixCacheProvider` layer any registered slot resolves through:
`this.cache.use('memcached')` (no dedicated `.memcached` getter, matching
every non-`redis`/`local` slot's own convention) returns a working
`ZanixMemcachedConnector` directly.

## `ZanixQLRUConnector`

Synchronous, `Map`-backed local LRU. `capacity` → `LOCAL_CACHE_MAX_ITEMS` env
var (default 50000). Evicts a single oldest entry once over capacity — one
eviction per insert past the limit, not a batch trim.

## Multi-layer provider (`ZanixCacheCoreProvider`)

```ts
await getCachedOrFetch('redis', key, fetcher) // first arg currently always the literal string 'redis'
await getCachedOrRevalidate('redis', key, fetcher, { softTtl: 45 }) // seconds; 45 is already the default
```

`this.local` (the QLRU layer) is always checked first, regardless of which
backend string is passed — the string argument names the *remote* layer to
fall through to, not a choice of "local vs remote." `getCachedOrRevalidate`
stores an envelope `{ value, timestamp }`, not the raw value, so it can
compute staleness against `softTtl` on read.

## `withLock`

```ts
await connector.withLock('key', async () => {/* exclusive per-key access */})
```

**Caution, don't assume this is a distributed lock**: it's single-process,
in-memory only — it serializes concurrent access *within one process*, and
provides zero coordination across replicas. The same internal keyed lock
manager backs `datamaster-database-and-models`' SQLite KV store's `withLock`
too — same caveat applies there. If you need cross-replica mutual exclusion,
this isn't it; look for a distributed-lock primitive instead (this package
doesn't provide one).

## Checklist before adding a new cached value

- [ ] Is `clear()` ever called against this Redis instance from anywhere in
      the app? If the instance is shared with anything else, that's a real
      risk, not a hypothetical one.
- [ ] Does the code assume `withLock` provides cross-replica exclusivity? It
      doesn't — verify the actual concurrency model needed before relying on
      it for correctness in a multi-replica deployment.
- [ ] Is `getCachedOrRevalidate`'s `softTtl` tuned deliberately, or left at
      the 45s default without considering whether that's actually
      appropriate for this value's real staleness tolerance?

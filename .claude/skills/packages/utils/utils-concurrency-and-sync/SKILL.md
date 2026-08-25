---
name: utils-concurrency-and-sync
description: Semaphore/LockManager for in-process concurrency control, nextCronDate for computing a cron expression's next fire time, and planCodeSync — the pure reconciliation function behind every code-to-storage sync in the ecosystem (seed/resync/leave-alone/orphan). Use when serializing concurrent access to a resource, computing a cron schedule, or reconciling static code entries against persisted ones.
---

Covers `@zanix/utils/helpers`'s concurrency primitives and `planCodeSync`
— the reconciliation function `@zanix/admin`'s templates-sync endpoint and
`@zanix/notifications`'s `LocalTemplateBackend` both apply. File:line
references point at `~/Documents/Development/ZanixLibraries/utils` — read
the real code there before assuming this summary is still accurate.

## Golden rule (token savings)

- `LockManager` defaults to exclusive (1 permit) per key — different keys
  still run fully in parallel, only same-key calls serialize. Check this
  before assuming a `LockManager` instance serializes everything it
  touches.
- `planCodeSync` is pure and does no I/O — it only computes a plan; the
  caller is responsible for actually applying `toSeed`/`toResync`/
  `toOrphan`.

## `Semaphore` and `LockManager`

```ts
import { LockManager, Semaphore } from 'jsr:@zanix/utils@[version]/helpers'

const sem = new Semaphore(3) // up to 3 concurrent holders
await sem.acquire()
try { /* ... */ } finally { sem.release() }

const locks = new LockManager() // 1 permit per key by default — exclusive
await locks.withLock('user:42', async () => { /* ... */ })
```

`Semaphore(permits)` — `acquire()`/`release()` around a fixed number of
concurrent holders; `release()` returns a boolean. `LockManager(
permitsPerKey?)` wraps a `Semaphore` per key — `withLock<T>(key, fn)`
acquires the key's own permit, runs `fn`, releases regardless of
success/failure. Default `permitsPerKey` is `1` (exclusive, a real mutex
per key) — two calls with **different** keys still run fully in parallel;
only calls sharing the same key ever wait on each other.

## `nextCronDate`

```ts
const next = await nextCronDate('0 */5 * * * *') // every 5 minutes, from now
const nextFrom = await nextCronDate('0 0 * * * *', someDate)
```

`nextCronDate(cronExpr, fromDate?)` — 6-field cron (`second minute hour day
month weekday`), the same expression shape `@zanix/asyncmq`'s
`registerCronJob` and `@zanix/app`'s scheduled `jobs` use. Returns
`undefined` for an invalid expression rather than throwing.

## `planCodeSync`: the reconciliation function behind code-to-storage sync

```ts
const plan = planCodeSync(staticEntries, existing)
// plan.toSeed    — code entries with no persisted record at all
// plan.toResync  — persisted entries whose code value changed, untouched since last sync
// plan.toOrphan  — persisted entries with no matching code entry
// (left alone)   — a manually-edited entry, or one with no sync history, always wins
```

`planCodeSync<V, Id>(staticEntries, existing, equals?)` — pure, no I/O.
Compares a package's static in-code entries against what's already
persisted, and classifies every persisted entry into exactly one outcome:
`toSeed` (brand-new, no persisted record), `toResync` (the code value
changed AND the persisted value hasn't been manually edited since the last
sync), or left untouched (a manual edit, or no sync history at all, always
wins over resyncing). Entries no longer present in `staticEntries` become
`toOrphan`.

This is the exact function `@zanix/notifications`'s `LocalTemplateBackend`
applies to reconcile its own code-defined templates against a database,
and the same one `@zanix/admin`'s `POST /templates/sync` endpoint applies
when pulling a remote service's template set — one reconciliation
algorithm, reused rather than reimplemented per consumer. A new
code-to-storage sync feature anywhere in the ecosystem should reuse
`planCodeSync` rather than hand-rolling equivalent seed/resync/orphan
logic.

## Checklist before adding concurrency control or a new sync feature

- [ ] Is `LockManager`'s default exclusive-per-key behavior actually the
      right shape — or does this use case need a higher `permitsPerKey`
      for controlled (not fully exclusive) concurrency?
- [ ] Does a cron-adjacent feature use the real 6-field expression shape
      `nextCronDate`/`registerCronJob` share, not a 5-field Unix-cron copied
      from elsewhere?
- [ ] Does a new code-to-storage sync feature reuse `planCodeSync` rather
      than reimplementing equivalent seed/resync/orphan reconciliation
      logic by hand?

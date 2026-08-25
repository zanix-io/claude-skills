---
name: asyncmq-dlq-reprocessing
description: registerDLQProcessor — the @zanix/asyncmq/dlq bridge that reprocesses @zanix/datamaster's DlqProvider entries via a cron job, DLQProcessorOptions, the claim/handler/complete/fail cycle, and testing it without a live broker. Use when adding or reviewing reprocessing for a business-level Dead Letter Queue entry. For DlqProvider's own lifecycle/model/payload-protection, see datamaster-dlq instead.
---

`@zanix/asyncmq/dlq` reprocesses `@zanix/datamaster`'s Dead Letter Queue
(`DlqProvider`) entries — items that failed in some business process
(payments, webhooks, jobs...) and were persisted for auditing/debugging/
retry. `DlqProvider` itself is a **passive store**: it never claims or
interprets entries on its own (see `datamaster-dlq` for its lifecycle,
model registration, and payload protection — not repeated here).
`registerDLQProcessor` is the active half: a thin wrapper over
`registerCronJob` (`asyncmq-scheduling-and-cron`) that actually drives
reprocessing. File:line references point at
`~/Documents/Development/ZanixLibraries/asyncmq` — read the real code there
before assuming this summary is still accurate.

**`registerDLQProcessor`/`DLQProcessorOptions` are `@zanix/asyncmq`'s own
symbols, cased differently on purpose — not a stale reference.** `@zanix/
datamaster`'s own DLQ symbols (`DlqProvider`, `DlqEntryAttrs`, etc.) were
renamed from all-caps `DLQ` to `Dlq` casing; `@zanix/asyncmq`'s
`registerDLQProcessor`/`DLQProcessorOptions` are a different package's
symbols and were untouched by that rename — don't "fix" them to `Dlq` casing.

## Golden rule (token savings)

- `handler` needs zero manual claim/lease/complete/fail bookkeeping — all of
  it is handled by the wrapper (see "What happens on every tick" below).
  Don't write defensive claim/release logic inside `handler`; it's already
  done before `handler` runs.
- Test a processor against a real Mongo connection and the cron/task
  registry directly (see the testing pattern below) — no live RabbitMQ
  broker is needed to verify the claim → handler → complete/fail cycle.

## This is a separate subpath, deliberately

```ts
import { registerDLQProcessor } from '@zanix/asyncmq/dlq'
```

Importing `@zanix/asyncmq/dlq` is the only way to pull in
`@zanix/datamaster`'s module graph from this package — the rest of
`@zanix/asyncmq` (`asyncmq-connector-and-subscribers`,
`asyncmq-worker-and-tasks`, `asyncmq-scheduling-and-cron`) never does, so an
app that doesn't use DLQ reprocessing never pays for it.

**This is distinct from RabbitMQ's own broker-native dead-letter mechanism**
(`ZanixCoreAsyncMQProvider.requeueDeadLetters`) — that one reroutes messages
the broker itself moved after exhausting delivery retries.
`DlqProvider`/`registerDLQProcessor` is a persisted, broker-agnostic
registry of *business-level* failures, useful even in an app that never
touches a message queue at all. Don't conflate the two when reviewing a
"dead letter" mention — check which system it actually refers to.

## Basic usage

```ts
import { registerDLQProcessor } from '@zanix/asyncmq/dlq'

registerDLQProcessor('payment.process', {
  name: 'reprocess-payment',
  schedule: '0,30 * * * * *', // every 30s
  handler: async function (entry) {
    const payments = this.providers.get(PaymentsRepository)
    await payments.retry(entry.payload)
  },
})
```

The first argument, `processType`, selects which `DlqProvider` entries this
processor is responsible for — the same value passed to
`DlqProvider.push({ processType })` on the producing side
(`datamaster-dlq`). Register one `registerDLQProcessor` call per
`processType` that needs reprocessing.

## What happens on every tick

Registering a processor really registers a cron job (named `dlq:<name>`
internally, via `registerCronJob`) whose handler, each tick:

1. Atomically claims one eligible entry for `processType` via
   `DlqProvider.claim()` — a no-op tick if nothing is eligible (`handler` is
   never called).
2. Runs `handler`, passed the claimed entry.
3. On success, calls `DlqProvider.complete()`.
4. On a thrown error (sync or async), calls `DlqProvider.fail()` with the
   error's `name`/`message`/`stack` — `DlqProvider` itself decides whether
   that moves the entry back to `'pending'` (attempts remain) or to a
   terminal `'failed'` state (`maxAttempts` reached).

**Caution — this throw-to-fail signal looks like `registerJob`'s
throw-to-retry signal but is governed by an entirely different system.** A
plain job/cron handler that throws gets retried per
`QueueMessageOptions.retryConfig.maxRetries` (`asyncmq-worker-and-tasks`). A
DLQ processor `handler` that throws instead routes through
`DlqProvider.fail()`, whose pending-vs-failed decision is `DlqProvider`'s
own `maxAttempts`, a completely separate counter from any
`retryConfig.maxRetries` on the underlying cron job. Don't reason about a
DLQ processor's retry behavior using `asyncmq-worker-and-tasks`'s
`this.context.attempt` pattern — it doesn't apply here.

## `DLQProcessorOptions` reference

| Option | Required | Default | Notes |
| --- | --- | --- | --- |
| `name` | yes | — | Becomes part of the cron job's own name (`dlq:<name>`) — must be unique across all cron jobs, same registry `registerJob`/`registerCronJob` share. |
| `schedule` | yes | — | `registerCronJob`'s own real `CronJobDefinitionBase['schedule']` field, `Pick`ed directly rather than redeclared, so this type can never silently drift from the real cron contract. |
| `isActive` | no | `true` | Same field as `registerCronJob`'s own `isActive`. |
| `processingQueue` | no | `'soft'` | `'soft' \| 'moderate' \| 'intensive'` — same weight semantics as any other cron/job (`asyncmq-worker-and-tasks`). |
| `handler` | yes | — | `(entry: DlqEntryAttrs) => Promise<unknown> \| unknown`, called with the claimed entry. |

## Testing a processor without a live broker

`registerDLQProcessor` only needs a real Mongo connection and the cron/task
registry — no RabbitMQ required to test the claim → handler →
complete/fail cycle directly:

```ts
import { ProgramModule } from '@zanix/server'
import { getTask } from 'utils/tasks.ts'
import { CRONS_METADATA_KEY } from 'utils/constants.ts'

const getRegisteredCronTask = (cronName: string) => {
  const [, jobDef] = ProgramModule.registry.get(CRONS_METADATA_KEY).find((
    [name],
  ) => name === cronName)
  return {
    task: getTask(jobDef.args.$taskId, jobDef.queue),
    args: jobDef.args.$args,
  }
}

// ... registerDLQProcessor(...), then invoke the real registered task directly:
const { task, args } = getRegisteredCronTask('dlq:reprocess-payment')
await task.call({ providers: { get: () => dlq } }, args)
```

See `src/@tests/functional/jobs/dlq.test.ts` in this package for the full
working pattern (claim → handler → complete, a throwing handler, and an
empty tick that never invokes the handler) — copy its shape for a new
processor's own test rather than re-deriving the registry-lookup dance above
from scratch each time.

## Checklist before adding a new DLQ processor

- [ ] Does `processType` match exactly what the producing side passes to
      `DlqProvider.push({ processType })` — a mismatch silently means this
      processor never claims anything?
- [ ] Is `handler` free of manual claim/lease/complete/fail bookkeeping —
      trusting the wrapper to handle it entirely?
- [ ] Is the DLQ-specific `maxAttempts`-based failure semantics (via
      `DlqProvider.fail()`) kept mentally separate from
      `retryConfig.maxRetries` on a plain job/cron?
- [ ] Does the new processor have a test following the real
      `dlq.test.ts` claim → handler → complete/fail pattern, not a live
      broker/cron-timer-dependent test?

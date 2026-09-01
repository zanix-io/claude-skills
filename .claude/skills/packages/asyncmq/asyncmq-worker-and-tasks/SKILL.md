---
name: asyncmq-worker-and-tasks
description: registerJob's customQueue-vs-processingQueue duality (what actually decides Job vs Task semantics, not the function name), the Worker Provider's runJob/runTask dispatch split, executeGeneralTask, and the two ways to run the external worker (@zanix/core vs. the standalone two-entrypoint pattern). Use when registering or dispatching a distributed job or an internal task, or setting up a worker process.
---

This is `@zanix/asyncmq`'s job/task execution layer. For publishing/consuming
a plain queue message (no job semantics), see
`asyncmq-connector-and-subscribers`. For recurring/delayed execution
(`registerCronJob`, `schedule`), see `asyncmq-scheduling-and-cron` — cron
jobs share this skill's `customQueue`/`processingQueue` routing duality, so
read this one first. File:line references point at
`~/Documents/Development/ZanixLibraries/asyncmq` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- **The routing field decides Job-vs-Task semantics, not which function you
  called.** Check `customQueue` vs `processingQueue` on the registration —
  don't infer durability/persistence from the fact that `registerJob` was
  used, since it registers both shapes (see below).
- Copy the closest existing `registerJob`/`registerCronJob` call as a
  template rather than re-deriving the field shape from prose — the two
  registration functions share this exact duality.

## `registerJob` registers both "Jobs" and "Tasks" — there is no separate `registerTask`

There is exactly **one** registration function, `registerJob`
(`modules/jobs/task.defs.ts`), used for both shapes the docs call "Job" and
"Task." What determines the result is which routing field you set, not the
function name:

```ts
import { registerJob } from 'jsr:@zanix/asyncmq@^0.8.0/jobs'

// Distributed JOB — durable, extra-process, needs the external worker running.
registerJob({
  name: 'my-custom-job',
  args: { message: 'hello custom queue' },
  customQueue: 'extra-process-queue',
})

// Internal TASK — ephemeral, internal-process, no external worker needed.
const MAX_RETRIES = 5
registerJob({
  name: 'my-moderate-task',
  args: { message: 'hello local moderate queue' },
  processingQueue: 'moderate', // one of 'soft' | 'moderate' | 'intensive'
  settings: { retryConfig: { maxRetries: MAX_RETRIES } },
  handler: async function (args: { message: string }) {
    const repository = this.providers.get(SomeRepository)
    try {
      await repository.markProcessing(args.message)
    } catch (error) {
      if (this.context.attempt >= MAX_RETRIES) {
        await repository.markFailed(args.message, error)
        return
      }
      throw error // re-throw so AsyncMQ retries with backoff
    }
  },
})
```

A duplicate `name` throws `InternalError` at registration time regardless of
which field was used — names are unique across the whole job registry, not
scoped per routing kind.

**Caution — `soft`/`moderate`/`intensive` name two completely different
queue systems, not one.** The same three names exist both as predefined
*internal* task queues (`processingQueue`, internal-process, ephemeral) and
as predefined *AMQP* job queues (also reachable via `processingQueue` in the
`customQueue`-less Job shape, extra-process, durable) — six real queues
behind three shared names. Which one a `processingQueue: 'moderate'` call
actually resolves to depends on the surrounding `Job`/`Task` context, not
just the string itself; don't assume "moderate" always means the same queue.

## The `handler`'s `this` context, and the retry pattern

`handler`'s `this` is bound to the execution context: `this.providers`/
`this.connectors`/`this.interactors` (the same getters an Interactor uses)
and `this.context` (`this.context.attempt` — the current retry attempt —
and `this.context.queue`). A handler that throws is retried up to
`settings.retryConfig.maxRetries` (default from `QueueMessageOptions`).
Checking `this.context.attempt` against that same limit, as in the example
above, is how a handler distinguishes "still retrying" from "this was the
last attempt" without maintaining its own counter — don't add a
separate manual retry counter when `this.context.attempt` already tracks it.

## Dispatching: `runJob` vs `runTask` — match the field you registered with

The Worker Provider exposes two distinct dispatch methods, and they are
**not** interchangeable — each expects the job to have been registered with
the matching routing field. Verified directly against
`modules/worker/provider.ts:231`: calling `runTask` on a job that has no
`processingQueue` throws `"The job '<name>' should not be executed using
runTask, as it lacks a defined processingQueue."`

```ts
class LedgerInteractor extends ZanixInteractor {
  async mint(amount: number) {
    // customQueue-registered job → runJob
    await this.worker.runJob('my-custom-job', {
      args: { amount },
      contextId: this.contextId,
    })
  }
}
```

```ts
// processingQueue-registered job → runTask
worker.runTask('my-moderate-task', {
  args: { message: 'Hello local!' },
  callback: (response) => console.log(response.response),
})
```

**Caution — `runTask` fails silently if `setTaskerUrl` was never called.**
It dispatches to a real internal worker thread whose bootstrap module must
be registered beforehand (`setTaskerUrl`, from `@zanix/asyncmq/worker`) —
without it, `runTask` logs an error and returns `false` instead of throwing
or running the task. `@zanix/core`'s `Zanix.startWorker()` handles this
automatically; a standalone setup must call `setTaskerUrl` itself (see
below) or task dispatch degrades silently.

### `runJob`'s `provider` option is forward-looking, not functional yet

`options.provider` lets a caller target a non-default AsyncMQ provider slot
— but only one broker provider slot (`'asyncmq'`) exists today (see
`asyncmq-connector-and-subscribers`'s caution on this), so there's nothing
real to pass yet. The option exists at the call site deliberately: a job's
*registration* stays broker-agnostic (the exact same registered job also
runs through `runTask`, which has no provider/broker concept at all), so
tying a job's identity to a specific broker at registration time would
break that symmetry.

## Generic tasks with no dependency injection: `executeGeneralTask`

Inherited from `@zanix/server`'s `ZanixWorkerProvider` base class — not
AsyncMQ-specific. Runs a plain function inside a default `WorkerManager` (3
workers by default), internal-process:

```ts
const invokeTask = worker.executeGeneralTask(fn, {
  metaUrl: import.meta.url, // required
  timeout: 5000, // optional
  callback: (response) => {
    if (response.error) console.error(response.error)
    else console.log('Result from task:', response.response)
  },
})

invokeTask()
```

Use for lightweight computations/transformations with no need for
`this.providers`/`this.connectors` — like the predefined internal queues,
this never requires the external worker.

## Running the worker process

Predefined AMQP job queues and custom extra-process queues need a real
running worker process. `@zanix/asyncmq/worker` is a library of bootstrap
building blocks (`registerExtraProcessQueues`, `setTaskerUrl`,
`initWorkerEntrypoint`, `workerFileTypes`, `registerInternalProcess`,
`baseProcessor`) — it is **not** a runnable script by itself.

**The easy way — `@zanix/core`** (the recommended entrypoint):

```ts
// worker.ts
import Zanix from '@zanix/core'
Zanix.startWorker()
```

```bash
deno run -A worker.ts
```

This registers AsyncMQ's extra-process queues, registers `@zanix/core`'s own
internal-process bootstrap module as the tasker URL (so `runTask`'s local
fallback works), loads the project's own connectors/handlers/defs, and keeps
the process alive.

**Building it yourself — two entrypoints, one shared lifecycle.** A
standalone setup (no `@zanix/core`) needs two pieces, each marking a
different execution mode but sharing the same `initWorkerEntrypoint` call:

```ts
// worker.ts — the main-thread bootstrap (run this: deno run -A worker.ts)
import {
  initWorkerEntrypoint,
  registerExtraProcessQueues,
  setTaskerUrl,
} from '@zanix/asyncmq/worker'

setTaskerUrl(import.meta.resolve('./internal-process.ts'))
await registerExtraProcessQueues()
await initWorkerEntrypoint(async () => {
  // ...register/scan this project's own jobs, providers, connectors here...
})
```

```ts
// internal-process.ts — the entrypoint runTask spawns as a real Worker thread
import { baseProcessor, initWorkerEntrypoint, registerInternalProcess } from '@zanix/asyncmq/worker'

registerInternalProcess()
await initWorkerEntrypoint(async () => {
  // ...re-register/scan the SAME jobs, providers, connectors as worker.ts —
  // this thread has its own isolated module registry, nothing carries over.
})

export const processor = baseProcessor
```

**Caution — the internal-process entrypoint has its own isolated module
registry.** A spawned Worker thread can't receive function references
across a `postMessage` boundary, so `internal-process.ts` must
independently re-run whatever job/handler registration `worker.ts` already
did. Forgetting to mirror a registration into both files is a real, silent
failure mode: the job exists in the main thread's registry but not the
internal-process thread's, so a `runTask` dispatch for it fails at runtime
even though registration "looked" complete.

`initWorkerEntrypoint` runs, in order: the optional `loadDependencies`
callback → the `onSetup`/`onBoot`/`postBoot` target-initialization phases (so
`this.providers`/`this.connectors` resolve correctly inside a handler) →
metadata cleanup. `loadDependencies` runs before any initialization phase
deliberately, so anything it registers is visible regardless of which phase
it belongs to — it can be omitted entirely if a task/job needs no
providers/connectors.

**Real-world reference**: `@zanix/core`'s own `Zanix.startWorker()` is built
exactly this way — `src/modules/worker.ts` (main-thread) calls
`registerWorkerTaskerUrl()` (= `setTaskerUrl` pointed at `tasker.ts`) then
`registerExtraProcessQueues()`; `src/modules/tasker.ts` calls
`registerInternalProcess()` then the identical `initWorkerEntrypoint(...)`
call. Only the mode-marking call differs between the two files — everything
else (loading the project's own dependencies, running the init lifecycle) is
shared.

### `ZANIX_WORKER_EXECUTION`

Managed automatically, informational only — never set it manually:

| Value | Meaning |
| --- | --- |
| `main-process` | Main application execution (default) |
| `extra-process` | Running inside an external worker (AMQP jobs) |
| `internal-process` | Running inside an internal worker (local tasks) |

## Checklist before registering or dispatching a job/task

- [ ] Is the routing field (`customQueue` vs `processingQueue`) chosen for
      the actual durability/persistence need — not assumed from the
      registration function's name?
- [ ] Does the dispatch call (`runJob` vs `runTask`) match the routing field
      the job was registered with?
- [ ] If this needs the external worker, is `setTaskerUrl` actually
      registered somewhere in the standalone bootstrap — not just assumed
      because `@zanix/core` normally handles it?
- [ ] If building a standalone (non-`@zanix/core`) worker setup, does the
      internal-process entrypoint re-register the exact same jobs/providers/
      connectors as the main-thread bootstrap?
- [ ] Does a handler that can fail check `this.context.attempt` against its
      own `maxRetries` before doing something irreversible on the last
      attempt, rather than retrying indefinitely by habit?

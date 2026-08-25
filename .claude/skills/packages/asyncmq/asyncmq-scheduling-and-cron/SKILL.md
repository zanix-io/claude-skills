---
name: asyncmq-scheduling-and-cron
description: The provider's schedule() method for delayed/future-dated messages, registerCronJob's DSL and cron execution metadata (info.cron), and how both integrate with the same retry/DLQ system as regular messages. Use when delivering a message in the future or registering a recurring job. Assumes asyncmq-worker-and-tasks's customQueue/processingQueue routing duality.
---

This covers `@zanix/asyncmq`'s time-based features: one-off delayed/future
delivery (`schedule`) and recurring execution (`registerCronJob`).
`registerCronJob` shares `registerJob`'s exact `customQueue`/
`processingQueue` routing duality and retry mechanics — see
`asyncmq-worker-and-tasks` for that (not repeated here). For plain
publish/consume without timing, see `asyncmq-connector-and-subscribers`.
File:line references point at
`~/Documents/Development/ZanixLibraries/asyncmq` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- `registerCronJob`'s field shape is `registerJob`'s field shape plus
  `schedule`/`isActive` — verify only the cron-specific fields against real
  source; trust `asyncmq-worker-and-tasks` for the shared routing/retry
  mechanics rather than re-deriving them here.
- A `date`/`delay` in the past throws `ApplicationError` at the call site —
  that's enough to catch a scheduling bug at the point it's introduced;
  don't add a separate defensive check before calling `schedule`.

## Scheduling a one-off delayed/future message

From inside a Zanix-managed class, `this.asyncmq.schedule(...)`. Outside
one, the same `ProgramModule.providers.get('asyncmq')` lookup
`asyncmq-connector-and-subscribers` documents for `enqueue`:

```ts
await asyncmq.schedule('email.send', { email: 'user@example.com' }, {
  delay: 60_000, // 1 minute
  isInternal: true,
})

// Or an absolute date:
await asyncmq.schedule('email.send', { email: 'user@example.com' }, {
  date: new Date('2025-01-01T10:00:00Z'),
  isInternal: true,
})
```

| Option | Type | Description |
| --- | --- | --- |
| `date` | `Date` | Absolute delivery date. Overrides `delay` if both are given. |
| `delay` | `number` | Delivery delay in ms (default `0`). |
| `isInternal` | `boolean` | Resolves the queue via the internal queue path mechanism. |
| `...options` | `QueueMessageOptions` | Standard publishing options (except expiration). |

All scheduled messages are encrypted, persisted, and delivered exactly once
at execution time — same encryption guarantee as any other AsyncMQ message
(`asyncmq-connector-and-subscribers`).

## Registering a cron job

```ts
import { registerCronJob } from 'jsr:@zanix/asyncmq@latest'

registerCronJob({
  name: 'minuteJob',
  isActive: true,
  customQueue: 'taskQueue', // or processingQueue + handler — see asyncmq-worker-and-tasks
  args: { foo: 'bar' },
  schedule: '0 */1 * * * *', // every minute
})
```

| Field | Type | Description |
| --- | --- | --- |
| `name` | `string` | Unique across **all** cron jobs (shares the job-name registry with `registerJob`). |
| `isActive` | `boolean` | Enables/disables the cron job. |
| `customQueue` / `processingQueue` + `handler` | see `asyncmq-worker-and-tasks` | Same routing duality as `registerJob` — determines durable/distributed vs. ephemeral/internal. |
| `args` | `MessageQueue` | Payload sent on each execution. |
| `schedule` | 6-field cron string (seconds precision) | Cron expression. |
| `settings` | `Omit<QueueMessageOptions, 'contextId' \| 'isInternal'>` | Optional publishing options. |

A duplicate `name` throws `InternalError` at registration time — same
uniqueness rule as `registerJob`, since both share one job registry.

## Cron execution metadata: `info.cron`

A cron-triggered message's `onmessage`/`onerror` receives extra metadata the
handler can use to distinguish a cron execution from a normal one:

```ts
async onmessage(data: any, info: MessageInfo) {
  console.log(info.cron) // { name, expression, nextExecution }
}

async onerror(data: unknown, error: unknown, info: ErrorInfo) {
  console.log('Requeued:', info.requeued)
  console.log('Cron job:', info.cron?.name)
}
```

## Error handling and retries

Cron jobs and scheduled messages integrate with the same retry system as any
other message — failed executions follow the same retry rules, may be
requeued or routed to DLQ, and `onerror` handlers receive full scheduling
metadata alongside the normal `info.requeued`. There is no separate
retry/backoff configuration specific to cron or scheduled messages; it's the
same `QueueMessageOptions`/`this.context.attempt` mechanics
`asyncmq-worker-and-tasks` documents.

## Checklist before adding a scheduled message or cron job

- [ ] Is the delivery timing expressed the simplest correct way — `delay`
      for "N ms from now," `date` only when an absolute wall-clock time is
      actually required?
- [ ] Does the cron `schedule` string have all 6 fields (seconds precision),
      not a 5-field Unix-cron expression copied from elsewhere?
- [ ] Is the routing field (`customQueue` vs `processingQueue`) chosen per
      `asyncmq-worker-and-tasks`'s duality, not assumed from context?
- [ ] Does a handler that needs to tell a cron-triggered run apart from a
      manually-dispatched one read `info.cron`, rather than inferring it
      from the payload shape?

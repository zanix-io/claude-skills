---
name: asyncmq-connector-and-subscribers
description: ZanixRabbitMQConnector, ZanixCoreAsyncMQProvider's enqueue/sendMessage API, the @Subscriber decorator + ZanixSubscriber base class, the queue-handler validation flow (data/info), payload encryption, and AMQP_URI-based connector auto-loading. Use when consuming or publishing to a queue directly, or defining a new queue handler class.
---

This is the connector/provider/subscriber layer of `@zanix/asyncmq` — the
RabbitMQ connection itself, publishing a message, and consuming one. For
registering distributed jobs/internal tasks (`registerJob`, the Worker
Provider), see `asyncmq-worker-and-tasks`. For delayed messages and cron
(`schedule`, `registerCronJob`), see `asyncmq-scheduling-and-cron`. For
`@zanix/datamaster`'s Dead Letter Queue reprocessing bridge, see
`asyncmq-dlq-reprocessing`. File:line references point at
`~/Documents/Development/ZanixLibraries/asyncmq` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- The `this.asyncmq` getter (inside any `@zanix/server` `CoreBaseClass`
  subclass — an Interactor, a Provider) is the idiomatic access path in
  real code. Reach for the `ProgramModule.providers.get('asyncmq')` lookup
  only when genuinely outside a Zanix-managed class — don't default to it
  out of caution.
- Verify one connector/provider method's real signature with a targeted
  grep (`modules/rabbitmq/provider/mod.ts`) rather than reading the whole
  provider file — the README's examples are accurate and current for the
  common calls (`enqueue`, `sendMessage`, `schedule`).

## The three real pieces, and what each owns

| Piece | Role | Real source |
| --- | --- | --- |
| `ZanixRabbitMQConnector` | AMQP connection/reconnection, lightweight channels with restricted operations, queue/binding/consumer declaration, `ack`/`nack` | `modules/rabbitmq/connector.ts` — extends `ZanixAsyncmqConnector` |
| `ZanixCoreAsyncMQProvider` | Initializes the connector, registers queues/handlers, validates payloads via `rto`, routes messages, retries/failures, publishing (`enqueue`/`sendMessage`), scheduling/cron | `modules/rabbitmq/provider/mod.ts` |
| `@Subscriber` + `ZanixSubscriber<Interactor>` | Registers a class as a queue handler; `onmessage(data, info)` is the one method every subscriber implements | `modules/subscribers/decorators/base.ts`, `modules/subscribers/base.ts` |

## Broker extensibility: replace-only by design, settled

`ZanixAsyncmqConnector` (`@zanix/server`, `connectors/core/asyncmq.ts`) is a
genuinely empty abstract class — zero methods, despite its own JSDoc
inviting "Kafka, MQTT, etc." implementations. `ZanixCoreAsyncMQProvider`
types its connector as the concrete `ZanixRabbitMQConnector`, not the
abstract base, and calls AMQP-specific methods (`createChannel()`) that
don't exist on the abstract contract at all. There is also only ONE
`'asyncmq'` core slot in the whole framework (`providers/core/all.ts`,
`connectors/core/all.ts`) — unlike `@zanix/auth`'s OAuth2 providers, which
coexist under separate keys (`'google-oauth2'`, `'github-oauth2'`).

**This is a settled architectural decision, not an open gap**: a new
broker (Kafka, SQS, Redis Streams) would need a full second provider
implementation against that broker's real semantics (comparable in scope
to building the queue/retry/DLQ/cron logic from scratch), and would
REPLACE RabbitMQ rather than coexist with it — no broker-selection
mechanism exists, and none is planned speculatively. Designing
coexistence (multiple slots, an abstraction layer that doesn't leak AMQP
specifics) is real, costly architecture work with no concrete second
broker driving it today — revisit only when an actual second-broker
requirement is proposed, as part of that work, not ahead of it. This is
also why no `asyncmq-builder` agent exists (mirroring `auth-builder`):
there's only one real broker instance to model a repeatable workflow
against.

## Defining a subscriber

```ts
import { Subscriber, ZanixSubscriber } from 'jsr:@zanix/asyncmq@latest'
import type { MessageInfo } from 'jsr:@zanix/asyncmq@latest'

@Subscriber({
  queue: 'email.send',
  Interactor: EmailInteractor,
  rto: EmailRto, // validate incoming message
})
class EmailSubscriber extends ZanixSubscriber<EmailInteractor> {
  async onmessage(data: { email: string }, info: MessageInfo) {
    await this.interactor.send(data.email)
  }
}
```

`@Subscriber` also accepts a bare route string (shorthand for `{ queue:
route }`), or a full config object for an `extra-process` custom queue:

```ts
@Subscriber({ queue: { topic: 'extra-process-queue', execution: 'extra-process' } })
export class _Subscriber extends ZanixSubscriber {
  protected async onmessage(args: { message: string }) {}
}
```

This `extra-process` shape is how a subscriber consumes from a genuinely
custom AMQP queue (as opposed to the predefined `soft`/`moderate`/`intensive`
job queues `registerJob`'s `customQueue` targets — see
`asyncmq-worker-and-tasks` for that distinction) — it still needs the
external worker process running, same as any other `extra-process` queue.

## The validation flow every message goes through

1. Message arrives from AMQP.
2. If encryption is enabled, decrypted.
3. Parsed as JSON.
4. Validated against the `rto` DTO passed to `@Subscriber`.
5. Delivered to `onmessage` only if valid.

Invalid payloads are logged and routed to DLQ — never silently dropped, and
never delivered to `onmessage` unvalidated. `onmessage`'s `info` parameter
carries delivery metadata (delivery tag, attempt count); for a cron-triggered
execution it additionally carries `info.cron` (see
`asyncmq-scheduling-and-cron`).

## Publishing: `enqueue`/`sendMessage`

From inside a Zanix-managed class, `this.asyncmq` is already the resolved
provider:

```ts
class LedgerInteractor extends ZanixInteractor {
  async mint(amount: number) {
    await this.asyncmq.enqueue('email.send', { email: 'user@example.com' }, {
      isInternal: true,
      contextId: this.contextId,
    })
  }
}
```

Outside a Zanix-managed class:

```ts
import { ProgramModule } from '@zanix/server'
import type { ZanixCoreAsyncMQProvider } from 'jsr:@zanix/asyncmq@latest'

const asyncmq = ProgramModule.providers.get<ZanixCoreAsyncMQProvider>('asyncmq')

await asyncmq.enqueue('email.send', { email: 'user@example.com' }, {
  isInternal: true,
  contextId: '',
})
await asyncmq.sendMessage('*', { message: 'hello queue' }, { contextId: '' }) // all queues
```

`enqueue` targets one named queue; `sendMessage('*', ...)` broadcasts to
every queue. `schedule` (delayed/future delivery) is the same provider's
method, covered fully in `asyncmq-scheduling-and-cron` rather than here — it
shares this provider but is conceptually a distinct feature (timing, not
routing).

## Encryption and connector auto-loading

- All outgoing messages are encrypted; all incoming messages are decrypted
  before validation. AES-based authenticated encryption (confidentiality +
  integrity), keyed by `DATA_AMQP_SECRET`.
- When `AMQP_URI` is present, the default connector and provider are
  automatically registered — no manual provider configuration needed for the
  common single-broker case.

| Variable | Description |
| --- | --- |
| `AMQP_URI` | RabbitMQ/AMQP connection URI. Its presence is what triggers auto-registration. |
| `DATA_AMQP_SECRET` | Symmetric key for encrypting/decrypting queue payloads. |

**Caution — an unset `DATA_AMQP_SECRET` doesn't fail, it silently degrades
confidentiality.** If `DATA_AMQP_SECRET` isn't configured, the provider falls
back to a hardcoded, publicly-known key (`modules/rabbitmq/provider/mod.ts`)
instead of throwing. It still logs a `logger.high` warning when this happens,
but message bodies — including anything parked in the dead-letter queue —
are not actually confidential until the env var is set.

**Caution — there is currently only one broker provider slot.** `'asyncmq'`
is one of `@zanix/server`'s reserved core slots, and it's singular today —
`@zanix/asyncmq` doesn't yet support registering a second simultaneous
broker provider under a different slot. `runJob`'s `options.provider` exists
as a forward-looking call-site override for when that changes (see
`asyncmq-worker-and-tasks`), but there's nothing real to pass there yet —
don't design around a multi-broker setup that doesn't exist.

## Checklist before adding/changing a subscriber or publish call

- [ ] Does the subscriber validate its payload via `rto`, rather than
      trusting `onmessage`'s `data` untyped?
- [ ] Is the queue name/config shape (bare route vs. `{queue, Interactor,
      rto}` vs. `{queue: {topic, execution: 'extra-process'}}`) the right one
      for whether this needs the external worker running?
- [ ] Is publishing done through `this.asyncmq` inside a Zanix-managed
      class, falling back to `ProgramModule.providers.get('asyncmq')` only
      when genuinely outside one?
- [ ] Does `contextId` get propagated on `enqueue`/`schedule`/`runJob` calls
      that originate from a real request, for correlation?

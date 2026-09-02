---
name: notifications-provider
description: NotifierProvider — the high-level send API (.email()/.sms()/.whatsapp()/sendMessage()), native WhatsApp provider templates, the mail trigger-action integration with @zanix/datamaster, and background-worker queuing. Use when sending a notification, wiring the datamaster mail trigger action, or reviewing queuing/worker behavior.
---

The recommended entrypoint for most apps, instead of reaching a connector
directly (see `notifications-connectors` for that layer). File:line
references point at `~/Documents/Development/ZanixLibraries/notifications` —
read the real code there before assuming this summary is still accurate.

## Golden rule (token savings)

- Trust `this.providers.get('notifications')`/`this.providers.get(NotifierProvider)`
  (both resolve the same registered instance — `@zanix/notifications` registers
  `NotifierProvider` under the `'notifications'` core-provider slot AND under its
  own class reference, the identical dual-registration `@zanix/auth`'s
  `ZanixAuthProvider` uses, see `auth-jwt-and-sessions`) for anything inside a
  Zanix-managed class. `new NotifierProvider()` is only for genuinely standalone
  usage outside the ecosystem's own lifecycle (see the `onDestroy()` note below) —
  reaching for it inside a handler/interactor bypasses the pooled, per-request
  `SCOPED` instance and its connector lifecycle for no reason.
- Copy the send-call shape from this skill's own examples rather than
  re-reading `NotifierProvider`'s source for routine sends — the API surface
  is small and stable.

## Sending

```ts
// Inside a Zanix-managed class (handler/interactor/etc.) — either form works:
const provider = this.providers.get('notifications')
// const provider = this.providers.get(NotifierProvider) — same instance, class form

await provider.email({
  to: 'recipient@example.com', subject: 'Welcome to Zanix',
  zanixTemplate: 'welcome', data: { buttonText: 'Click here' },
})
```

`.email()`/`.sms()`/`.whatsapp()` are typed convenience wrappers over the
generic `sendMessage(notifier, message, options)` — prefer them when the
channel is known statically, since each is typed against that channel's own
template registry. A message's content is either plain `content` text or
`zanixTemplate` + `data` — **mutually exclusive, a type error to set both**.
`subject` only applies to (and is required for) email; SMS/WhatsApp ignore
it, along with `from`/`date`.

## `.whatsapp()` dispatches automatically between two systems

```ts
// Renders `otp` via Handlebars, sent as freeform text → sendMessage()
await provider.whatsapp({ to, zanixTemplate: 'otp', data: { code: '123456', ttl: 5 } })
// Native provider template → sendTemplate()
await provider.whatsapp({ to, templateName: 'hello_world', templateLanguage: 'en_US' })
```

Whichever shape the message carries (`zanixTemplate`/`content` vs.
`templateName`/`contentSid`) decides which underlying call fires — see
`notifications-connectors` for the two systems this straddles. `sendTemplate()`
can also be called directly with the same `useWorker`/error-wrapping
behavior described below.

## The `mail` trigger action contract

`@zanix/datamaster`'s built-in `mail` trigger action only declares
`to`/`subject` itself — the rest (which template, what data) is **this
package's own contract**, not datamaster's:

```ts
import { sendMailTriggerNotification } from '@zanix/notifications'

await sendMailTriggerNotification(provider, {
  to: 'recipient@example.com', subject: 'Welcome to Zanix',
  body: { template: 'welcome', data: { buttonText: 'Click here' } },
})
```

`body.template` is authored dynamically (e.g. a trigger's own database-editable
config), so — unlike `.email()`'s `zanixTemplate` — it's **not statically
checked** against the built-in registry; an unregistered name surfaces as a
runtime error, not a compile-time one. Keep this in mind when reviewing a
trigger definition referencing a template by name — see `datamaster-triggers`
for the trigger side of this contract.

## Registering a new trigger action job

`mail` isn't special-cased — it's this package's own instance of
`@zanix/datamaster`'s general `registerTriggerActionJob` mechanism, and any
new channel-level trigger action this package gains later follows the same
shape. `providers/trigger-mail.core.ts` is the real precedent to copy:
`registerMailTriggerJob()` self-registers `'mail'`'s job descriptor
(`{name, processingQueue, handler}`, handler typed against a minimal
`{providers}` context, no direct `@zanix/asyncmq` dependency) via
`registerTriggerActionJob` imported from `@zanix/datamaster`, and this runs
as a module-load side effect from the package's own `/core` entrypoint —
loaded by `@zanix/core`'s `defineCoreMetadata()`, so it reaches both the main
process and the worker process with zero consumer-side setup. `@zanix/core`
only drains the descriptor and performs the real `@zanix/asyncmq`
`registerJob()` call; it never authors the handler's logic.

**This eager module-load-side-effect idiom is safe here on its own** — the
descriptor it builds doesn't read a binding from a file importing back into
this one. It's the same general idiom (a real function call at module top
level, not just a declaration) that caused a real, shipped crash elsewhere
in this package (`email/defs.ts`'s `registerSmtpConnector()`) when paired
with an accidental import cycle — see `zanix-dependency-direction`'s
"intra-package circular imports with a top-level side effect" section
before copying this shape into a new trigger action job whose descriptor
DOES read something from a file that could import back into
`trigger-<action>.core.ts`.

**The ownership check before registering a new one**: does the action's real
work use a capability this package already has — `NotifierProvider`,
`sendMessage()`, a connector? If yes, this package self-registers the
working handler here, the same shape as `mail`. If the action's real work
needs something this package doesn't already depend on, that registration
belongs wherever that capability actually lives — not here, even if the
trigger conceptually sounds notification-adjacent. See
`datamaster-triggers` for the full two-step mechanism (the registry vs. the
`@zanix/core` drain step, and why a package's own self-registration and
`@zanix/core`'s drain never double-register the same action kind) and
`zanix-dependency-direction` for why this direction — the capability's owner
registers into datamaster's registry, never the reverse — holds in general.

## Queuing with a background worker

```ts
await provider.email({ /* ... */ }, { useWorker: 'persisted' })
// or, for a per-message callback/timeout:
await provider.email({ /* ... */ }, {
  useWorker: { mode: 'one-time', callback: (response) => { if (response.error) console.error(response.error) } },
})
```

`'one-time'` (a fresh worker per flush) or `'persisted'` (the app's pooled
`'worker'` core provider, via `@zanix/server`'s `dispatchWorkerTask`) —
`'persisted'` **falls back to `'one-time'` automatically outside a booted
Zanix Core application**, so it's always safe to request regardless of
runtime context.

**Queued messages are flushed by `onDestroy()`** — inside the Zanix
ecosystem this runs automatically when the provider instance is torn down at
request end. Standalone (outside the ecosystem's own lifecycle), call it
yourself: `provider['onDestroy']()`. The bracket-notation access is
deliberate — this isn't meant to be called directly under normal operation;
reach for it only in genuinely standalone usage, not as a routine pattern
inside an app already running through the Zanix lifecycle.

**All queued messages — potentially spanning several channels, potentially
mixing `'one-time'`/`'persisted'` requests — flush from a single background
worker invocation**, each re-resolving its own channel's connector. When a
batch mixes modes, **`'persisted'` wins for the whole flush** if any one
queued message asked for it — never silently downgraded to `'one-time'`
because another message didn't request it. A send failure (inline or from
the background worker) is always re-thrown as an `InternalError` with code
`'NOTIFICATIONS_DISPATCH_FAILED'`, with the original error attached as
`.cause` — check `.cause`, not the top-level error, for the real failure
reason.

## Checklist before sending or reviewing a send call

- [ ] Is the provider accessed via `this.providers.get('notifications')`/
      `this.providers.get(NotifierProvider)` inside a Zanix-managed class,
      rather than `new NotifierProvider()` (standalone-only) or a hand-rolled
      connector call?
- [ ] Is `content` or `zanixTemplate`+`data` used, never both (a type error
      either way, but worth confirming the intent matches)?
- [ ] For a `mail` trigger action's `body.template`, is the referenced
      template name confirmed to exist — since this path isn't statically
      checked, an unregistered name only fails at runtime?
- [ ] If queuing, is `onDestroy()` actually reachable in this runtime context
      (automatic inside the Zanix ecosystem, manual otherwise) — a queued
      message with nothing ever flushing it silently never sends?
- [ ] On a caught send failure, is `.cause` inspected for the real error,
      not just the wrapping `InternalError`
      (`'NOTIFICATIONS_DISPATCH_FAILED'`)?

---
name: notifications-connectors
description: @zanix/notifications' per-channel connectors — SmtpClient, SmsClient, WhatsappClient — the pluggable provider-adapter pattern (consumer-side and shipping a new built-in adapter), and native WhatsApp Business template messages vs. this package's own Handlebars templates. Use when configuring a channel directly, plugging in a custom delivery provider, adding a new built-in vendor adapter, or reviewing connector-level behavior.
---

Covers the connectors directly — for the high-level send API most apps use
instead, see `notifications-provider`. File:line references point at
`~/Documents/Development/ZanixLibraries/notifications` — read the real code
there before assuming this summary is still accurate.

## Golden rule (token savings)

- Confirm a specific connector's config shape or env var against this
  skill's tables directly — don't re-derive it by reading the connector's
  full source for routine configuration.

## Three connectors, one shared base

`SmtpClient` (email), `SmsClient` (SMS), `WhatsappClient` (WhatsApp) all
extend `ZanixNotifierConnector` (`send(message): Promise<void>`, plus the
usual `@zanix/server` connector lifecycle: `initialize()`/`close()`/
`isHealthy()`). Each delegates actual delivery to a pluggable **provider
adapter** rather than being hardcoded to one vendor — a built-in adapter by
default, or your own.

| Connector | Built-in adapter(s) | Registers from |
| --- | --- | --- |
| `SmtpClient` | (direct SMTP, no separate adapter concept) | `SMTP_HOST`/`SMTP_PORT`/`SMTP_USER`/`SMTP_PASSWORD` |
| `SmsClient` | `TwilioSmsAdapter`, `VonageSmsAdapter` — auto-detected from whichever's vars are set; both set requires `SMS_PROVIDER=twilio\|vonage` or it throws | `TWILIO_ACCOUNT_SID`/`TWILIO_AUTH_TOKEN`/`TWILIO_FROM_NUMBER`, or `VONAGE_API_KEY`/`VONAGE_API_SECRET`/`VONAGE_FROM` |
| `WhatsappClient` | `MetaCloudWhatsappAdapter`, `TwilioWhatsappAdapter` — auto-detected from whichever's vars are set; both set requires `WHATSAPP_PROVIDER=meta\|twilio` or it throws | `META_PHONE_NUMBER_ID`/`META_ACCESS_TOKEN`, or `TWILIO_ACCOUNT_SID`/`TWILIO_AUTH_TOKEN`/`TWILIO_WHATSAPP_FROM` |

Importing `@zanix/notifications/core` registers whichever channels have their
required env vars set — each channel registers **independently**; only
`SMTP_*` set still registers `SmtpClient`, with `SmsClient`/`WhatsappClient`
simply skipped.

## Plugging in a different provider

Every connector's adapter is a tiny, swappable contract:

```ts
import type { SmsProviderAdapter } from '@zanix/notifications'

const myAdapter: SmsProviderAdapter = {
  send: async (message) => {/* call your provider's API */},
}
SmsClient.config = { adapter: myAdapter }
```

Same pattern for `WhatsappProviderAdapter`. No connector-level change needed
for a new vendor (Vonage — now also a real, shipped SMS option, see below —
AWS SNS, etc.) — implement the contract, set it as `config.adapter`.

## Shipping a new built-in provider adapter (not a consumer's own custom one)

The section above covers a *consumer* plugging in their own adapter at
runtime — no package change needed. Making a new vendor a first-class,
**shipped** option (the way `TwilioWhatsappAdapter` sits alongside
`MetaCloudWhatsappAdapter`) is a different, in-scope `notifications-builder`
task with a real, repeatable shape — `whatsapp/{meta,twilio,connector,defs}.ts`
is the precedent to copy, not something to design fresh:

1. **The adapter class** (`sms/twilio.ts`/`whatsapp/meta.ts`/`whatsapp/twilio.ts`
   are the templates) — implements the connector's provider-adapter contract
   (`SmsProviderAdapter`/`WhatsappProviderAdapter`, just `send()` plus
   whatever the contract requires), typically extending `@zanix/server`'s
   `RestClient` when the vendor is an HTTP API. Lives in the channel's own
   directory (`sms/`/`whatsapp/`), not a shared "adapters" folder — each
   channel owns its adapters.
2. **Env-var-gated selection in that channel's `defs.ts`** — `hasXEnv()`
   guard functions plus a `resolve<X>Provider()` function that auto-detects
   the provider from whichever single vendor's vars are set, and throws an
   `InternalError` if more than one vendor's vars are set with no explicit
   `<CHANNEL>_PROVIDER` selector (WhatsApp's `resolveWhatsappProvider` is the
   template — see `whatsapp/defs.ts`), building
   `WhatsappClient.config = { adapter: new NewAdapter({...}) }` from that
   vendor's own env vars. A new built-in vendor means adding it to that
   resolver's ambiguity check too, not just its own `hasXEnv()` guard.
   **Caution, confirmed real for `SmtpClient`'s own `defs.ts`**: this
   `Client.config = {...}` assignment is an eager, module-load-time side
   effect (it runs immediately at the top level of `defs.ts`, not inside a
   deferred call). If `defs.ts`, `connector.ts`, and any other file this
   assignment's values come from end up forming an import cycle (a shared
   constant round-tripping between them, for example), this exact line can
   throw `ReferenceError: Cannot access '<Client>' before initialization` —
   see `zanix-dependency-direction`'s "intra-package circular imports with a
   top-level side effect" section for the full precedent (this is the real
   bug that section documents) and the checklist to run before shipping a
   new channel/vendor whose `defs.ts` splits state across more than one
   file.
3. **The connector class itself (`connector.ts`) does *not* change** — it
   already resolves `this.#config.adapter ?? <built-in default>`; adding a
   second or third built-in option is `defs.ts`'s job (which adapter to
   *construct*), not a connector-level branch.
4. **This skill's own "Three connectors" table and provider-selection
   prose**, updated in the same change, matching "Docs move in the same
   change" (`notifications-builder`) — a new built-in adapter a newcomer
   should know about, not buried.

**Check the vendor's actual failure mode before assuming `RestClient`'s
automatic `HttpError`-on-non-2xx is enough** — Twilio fails at the transport
layer (a bad request comes back non-2xx, `RestClient` throws on its own),
but that's not universal: Vonage's classic SMS API always returns `HTTP 200`
and reports success/failure only in the JSON body's `messages[].status`
(`"0"` = accepted, anything else = rejected) — `VonageSmsAdapter.send()`
has to parse the body and throw manually for this reason. Verify this
against the real vendor's docs per-adapter, don't assume every REST-based
provider fails the same way Twilio does.

A whole **new channel** (a 4th beyond email/SMS/WhatsApp) stays out of
`notifications-builder`'s scope regardless — this recipe is for a new
*vendor* inside an existing channel, never a reason to treat a new channel
as the same kind of task.

## `SmtpClient`: pooling and registration lifetime

- **`SMTP_POOL_SIZE > 1`** switches from a fresh connection per request to a
  shared pool of persistent, authenticated connections — worth it since the
  SMTP handshake (`EHLO`/`AUTH LOGIN`) is the expensive part of every send.
  A connection the remote silently closed while idle is detected and
  replaced automatically.
- **`SmtpClient` is registered `SCOPED`, never `SINGLETON`** — SMTP is a
  single-socket, one-command-at-a-time protocol; sharing one instance across
  concurrent requests would interleave unrelated commands/responses on the
  same connection. Pooling is what avoids paying a fresh handshake per
  request without needing a shared instance — don't conflate the two.

## `WhatsappClient`: freeform text vs. native provider templates

`WhatsappClient.send()` only ever sends **freeform text** — deliverable
only within WhatsApp's 24h customer-service session window. Starting a
conversation outside that window needs a **pre-approved Business template
message**, a distinct capability via `sendTemplate()`:

```ts
// Meta Cloud API's shape
await client.sendTemplate({ to, templateName: 'otp_code', templateLanguage: 'en_US', templateParams: ['123456'] })
// Twilio's Content API shape
await client.sendTemplate({ to, contentSid: 'HX229f...', contentVariables: { '1': '123456' } })
```

**This is unrelated to this package's own Handlebars `zanixTemplate`
mechanism** (see `notifications-templates`) — `zanixTemplate` always means
"render locally and send as freeform text," never a WhatsApp Business
template. Don't conflate the two systems; they solve different problems and
neither substitutes for the other.

If both Meta's and Twilio's WhatsApp env vars are set with no
`WHATSAPP_PROVIDER` selector, `resolveWhatsappProvider()` **throws**
`InternalError` rather than picking a winner — set `WHATSAPP_PROVIDER=meta`
or `WHATSAPP_PROVIDER=twilio` to disambiguate (same shape for SMS via
`SMS_PROVIDER=twilio|vonage`). `TWILIO_WHATSAPP_FROM` is deliberately a
separate variable from `SmsClient`'s `TWILIO_FROM_NUMBER` — a
WhatsApp-enabled Twilio sender is typically a different number than the
plain SMS one, even under the same account.

## Caution: Twilio's route normalization can lowercase a case-sensitive path segment

`TwilioSmsAdapter`/`TwilioWhatsappAdapter` extend `@zanix/server`'s
`RestClient`, which normalizes every request path through `@zanix/helpers`'
`cleanRoute()` — **including lowercasing the dynamic `accountSid` URL
segment**. Twilio's real endpoint has case-sensitive path segments. This is
a documented, real caveat in the package's own docs, not a hypothetical —
verify against a real Twilio account before relying on this in production;
don't assume the adapter's request path matches Twilio's expected casing
without checking.

## Checklist before touching a connector

- [ ] Is the channel's registration actually independent — does setting only
      one channel's env vars leave the others correctly un-registered,
      rather than assuming all-or-nothing?
- [ ] For a custom provider, does the adapter satisfy the full contract
      (`send`, and any other required method) rather than a partial
      implementation that happens to compile?
- [ ] If touching `SmtpClient`, does anything assume a shared/singleton
      instance across concurrent requests? That's not how it's registered.
- [ ] If touching Twilio's adapters, has the real request path been verified
      against a live Twilio account — not just assumed correct from reading
      the adapter's source?
- [ ] Shipping a new built-in adapter, not a consumer-side one? Confirm it's
      a new *vendor* for an existing channel (in scope), not a new *channel*
      (out of scope) — and that `defs.ts`'s `resolve<X>Provider()` throws on
      an unresolved ambiguity (more than one vendor's vars set, no explicit
      `<CHANNEL>_PROVIDER` selector) rather than silently picking a winner.
- [ ] Does the channel's `defs.ts`/`connector.ts` (and any other file the
      `Client.config = {...}` assignment's values touch) form an import
      cycle? If so, does `defs.ts`'s top-level registration call read a
      binding from a file still mid-evaluation in that cycle — the real,
      already-shipped `SmtpClient` crash (see the caution above and
      `zanix-dependency-direction`'s "intra-package" section)?

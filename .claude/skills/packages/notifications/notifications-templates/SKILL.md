---
name: notifications-templates
description: @zanix/notifications' Handlebars template system — built-in per-channel registries, compiled-ahead-of-time rendering, and how to add a new base template (owning its own .hbs). Use when rendering a template directly, or adding a new base template. For templates that inherit another template's content instead of owning their own, see notifications-template-inheritance. For database-backed/remote template resolution, see notifications-template-storage-modes.
---

Covers the template system itself — for how `zanixTemplate` selects one of
these when sending a message, see `notifications-provider`. File:line
references point at `~/Documents/Development/ZanixLibraries/notifications` —
read the real code there before assuming this summary is still accurate.

## Golden rule (token savings)

- Copy an existing sibling template's file layout (`main.hbs`/`schema.ts`/
  `styles.css`) when adding a new one, rather than re-deriving the shape from
  this skill's prose each time.
- Run `deno task build-handlebars` once per batch of template changes, not
  after every single `.hbs` edit — it recompiles everything and regenerates
  `CODE_TEMPLATES`, not just the one file touched.

## Rendering is compiled ahead of time, never parsed at runtime

Handlebars templates compile into plain JS modules at build time — no
template parsing happens when a message is actually sent. This is why a new
`.hbs` isn't usable until `deno task build-handlebars` runs (see "Adding a
base template" below).

## Built-in registries

| Channel | Registry export | Templates |
| --- | --- | --- |
| Email | `transactionalTemplates` | `welcome`, `generic`, `password-changed`, `password-recovery`, `login-otp`, `new-login`, `data-table` |
| SMS | `smsTemplates` | `generic`, `otp`, `new-login` |
| WhatsApp | `whatsappTemplates` | `generic`, `otp` |

Each channel's registry is independent — SMS and WhatsApp each have their
own `generic`/`otp`, unrelated to email's. `generic` (all three channels)
takes at minimum `content: string`; email's also accepts `title`,
`buttonText`, `buttonLink`, `message`, `footer`, and an `html`/`styles`
override. `otp` (SMS/WhatsApp only — there's no email `otp`; use `login-otp`
or `password-recovery` instead) takes `code`, `ttl` (minutes), and an
optional `app` name.

## Rendering directly

```ts
import { transactionalTemplates } from '@zanix/notifications'
const html = await transactionalTemplates.welcome({ buttonText: 'Click here' })

// or the lower-level path, bypassing the registry layer entirely:
import { execTemplate } from '@zanix/notifications'
const html = await execTemplate('email/generic', { title: 'Hi', content: 'Welcome aboard!' })
```

Useful outside of `NotifierProvider` — previewing rendered content, or
embedding it somewhere other than a notification send.

## Adding a base template (owns its own `.hbs`)

**Maintainers only** — for a template that inherits another's content
instead, use `notifications-template-inheritance`'s procedure; picking the
wrong guide is the easiest way to end up with a template that renders fine
in code but is invisible to (or breaks) database-backed mode.

1. Create `handlebars/{channel}/{name}/main.hbs`, `schema.ts` (a Zod schema
   describing the data it accepts — also the source of its exported
   `*TemplateSchema` type), and `styles.css` (used by email's wrapping
   layout; still required, even empty, for plain-text SMS/WhatsApp — the
   build pipeline always injects `data.styles.css`).
2. Run `deno task build-handlebars` to compile every `main.hbs` into a
   `main.js` module — this is what `execTemplate()` actually imports at
   runtime. **This same task also regenerates `db/code-templates.generated.ts`'s
   `CODE_TEMPLATES`** directly from the `main.hbs` files it just found and
   compiled — nothing to register by hand here anymore; commit the
   regenerated file alongside the new `.hbs`/`schema.ts`/`styles.css`, the
   same way the compiled `main.js` is already committed.
3. Add a transactional wrapper function (see `src/modules/templates/transactional/`)
   calling `execTemplate('{channel}/{name}', data)`, and add it to that
   channel's registry object (`transactional/sms.ts`, `whatsapp.ts`, or
   `email/mod.ts`'s `templates` object) — the registry key is the real
   `zanixTemplate` string a caller passes.

## Checklist before adding a new base template

- [ ] Confirmed this template genuinely owns its own content, rather than
      transforming its input and rendering through an existing template
      (that's `notifications-template-inheritance`'s workflow instead)?
- [ ] Does `styles.css` exist even for a plain-text SMS/WhatsApp template —
      the build pipeline always injects it, regardless of channel?
- [ ] Was `deno task build-handlebars` run, and is the regenerated
      `db/code-templates.generated.ts` committed alongside the new
      `.hbs`/`schema.ts`/`styles.css` and the compiled `main.js`?
- [ ] Is the transactional wrapper added to the right channel's registry
      object, under the exact string callers will pass as `zanixTemplate`?
- [ ] Does the new `.hbs` avoid embedding any literal UI-chrome text (column
      headers, button labels, section headings) directly in the markup?
      Every base template in this package renders 100% caller-supplied
      copy — `title`/`content`/`footer`/etc. are always data, never fixed
      strings baked into the `.hbs`. `data-table`'s column headings
      (`Description`/`Qty`/`Subtotal`/`Total`) shipped hardcoded in English
      at first, breaking this convention, and needed a follow-up `labels`
      field (all optional, English defaults, same `defaultStyles`-merge
      shape `styles` already uses) to fix — check for this case before it
      ships, not after: any table/list-shaped template with its own
      generated headings needs the same `labels` treatment from day one.

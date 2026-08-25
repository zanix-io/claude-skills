---
name: notifications-template-inheritance
description: Templates that render through another template's content (parent) instead of owning their own .hbs — welcome/password-changed/password-recovery/login-otp/new-login (email) and otp/new-login (SMS/WhatsApp) all work this way. Use when adding a derived template, updating one whose content should follow its parent, or debugging why an edit to a shared template didn't propagate. Only matters once database-backed templates are enabled — invisible in pure code mode.
---

For the base template system this builds on (`name`/`hash`, sync rules,
multi-instance behavior), see `notifications-templates` and
`notifications-template-storage-modes`. This mechanism only matters once
database-backed templates are enabled — in pure code mode each derived
template is just a thin wrapper calling `execTemplate('{channel}/generic',
...)` directly, invisible as a distinct concept. File:line references point
at `~/Documents/Development/ZanixLibraries/notifications` — read the real
code there before assuming this summary is still accurate.

## Golden rule (token savings)

- Before editing a derived template's own record, check **whether it has
  real `hbs` of its own yet** (one grep/read of that record) — that single
  fact decides which of this skill's rules apply; don't reason through every
  rule for every edit.

## How it works

Once database-backed templates are enabled, each derived template
(`welcome`, `password-changed`, `password-recovery`, `login-otp`, `new-login`
for email; `otp`, `new-login` for SMS/WhatsApp) gets a "fallback" record
seeded alongside `generic`
itself — `source: 'database'`, no `hbs` of its own, a `parent: 'generic'`
field. `resolve()` walks `parent` (applying that name's registered data
transform at each hop) until it finds a record with real content:

```ts
// Editing the shared parent is enough — no need to also edit welcome/login-otp/etc.
await Model.updateOne({ channel: 'email', name: 'generic' }, { $set: { hbs: '...', hash: generateUUID() } })
```

- **`parent` is only consulted while a record has no `hbs` of its own.** The
  moment an admin gives a fallback record real content directly, it renders
  from that content instead — independent of `generic` from then on.
- **`parent` can be any template name**, with no code counterpart required —
  an admin can re-point `otp`'s `parent` at a brand-new `auth` record
  (itself `parent: 'generic'`, no code counterpart), and `otp → auth →
  generic` resolves and stays live-editable exactly like the single-hop
  case.
- **The chain can be more than one hop.** A cycle, or a hop pointing at a
  missing/inactive record, safely terminates the walk and falls back to the
  **original** template's code path — not whichever hop it stopped at.
- **A data transform only ever applies to the original `zanixTemplate`
  name's registered transform**, never re-applied at each hop — an
  intermediate hop with no code counterpart just passes its received data
  straight through unchanged.
- **A brand-new template with no code counterpart at all** can declare its
  own `parent` directly via a CRUD API and get the same fallback behavior,
  with zero code changes — as long as it's rendered with data already
  matching the parent's shape.

## How to update a derived template — pick the right target

- **To change what it currently inherits** (every email falling back to
  `generic` should look different): edit **`generic`'s** (or whichever
  ancestor's) `hbs`/`hash` — propagates automatically, no restart, no touch
  to the derived template's own record.
- **To give this one template independent content**, breaking it from its
  parent: set `hbs` (and `hash`) directly on **its own** record — the very
  next `resolve()` call sees real content and stops walking to `parent`,
  immediately, no restart, no "promotion" step needed (promotion only
  matters when *code* later adds a `.hbs` for this name — see below).
- **Don't set `hash` on a derived template's own record "to force a
  refresh"** — until it has real `hbs` of its own, `hash` is never read for
  cache invalidation at all. Only the ancestor's `hash` matters. Setting it
  anyway is a no-op, not a mistake that breaks anything, but it won't do
  what you think.

## Deleting or deactivating a fallback record — the one case that doesn't self-heal

The chain only ever starts by reading the requested template's **own**
record — if that record doesn't exist (or isn't `active`), `resolve()` falls
straight to the compiled code version, **ignoring any edit made to the
ancestor in the meantime**.

- **Delete, same process, no restart**: renders code-compiled until the
  process restarts.
- **Delete, then restart**: boot-time sync sees it missing and re-seeds a
  fresh fallback stub — the chain resumes.
- **Deactivate (`active: false`)**: **it never self-heals.** The sync step
  only checks whether *any* record exists for `{channel, name}` — an
  inactive one still counts as existing, so it's never recreated or
  reactivated automatically. It stays on the compiled code version until an
  admin reactivates that specific record by hand. This is the one deletion
  path in this whole mechanism with no automatic recovery — worth flagging
  explicitly if a review encounters a deactivated fallback record, since the
  usual "it'll heal on next sync" assumption doesn't hold here.

## Promoting a derived template to a base template

Giving a former derived template real `.hbs`/`schema.ts`/`styles.css` (per
`notifications-templates`'s procedure) creates a real collision: a database
record already exists for `{channel, name}` (the fallback stub), and the
next sync wants to seed a **code** record under the same unique-indexed key.
**Naively inserting a second document throws a duplicate-key error and takes
the entire sync down with it** — confirmed against a real MongoDB instance
while this feature was built, not a theoretical concern. The sync step
handles this by checking for a same-named collision before seeding:

- **No content of its own** (the common case — a fallback stub, `hbs`
  absent): **promoted in place** — updated to `source: 'code'` with real
  `hbs`/`hash`, never inserted as a duplicate.
- **Already has real content** (an unlikely but possible naming collision
  with a genuine database-only template): **left untouched**, code seed
  skipped, a warning logged — never silently destroying existing content.
  Rename one side to resolve the collision; nothing automates that choice,
  since guessing which side should win would be reckless.

## Cache invalidation follows the content owner, not the lookup name

`TemplateProvider`'s compiled-render cache is keyed by the record that
**owns** the content — the ancestor the chain lands on, never the derived
template that started the walk:

- Editing a derived template's own `hash` field (on its content-less
  fallback record) has **no effect on anything** — it's never read for
  cache invalidation, since that record never reaches the compile step at
  all.
- Once a fallback record is **promoted** and owns real `hbs`, its own `hash`
  starts mattering the normal way, like any code-backed or database-only
  template.

## Adding a derived template

**Maintainers only** — for a base template (owns its own `.hbs`), use
`notifications-templates`'s procedure instead.

**The same `name` can be declared independently in more than one channel** —
e.g. a `new-login` security notification derived in both `email` and `sms`,
each from that channel's own `generic`. `{channel, name}` is the real key
everywhere this matters (`DERIVED_TEMPLATES`, `resolve()`'s chain walk,
seeding) — there's no cross-channel registration step, no shared name
registry to collide against, and no need to coordinate the two channels'
entries beyond picking the same string for both by convention. Each
channel's own `transactional/*` file's `derivedTemplates` array is where its
own entry belongs, exactly like any other derived template in that channel.

1. Write a standalone transform function — `(data: YourSchema) =>
   ParentSchema` — in the relevant `transactional/*` file, plus a thin
   wrapper calling `execTemplate('{channel}/{parent}', yourTransform(data))`
   through it. **Keep the transform exported and standalone, not inlined**
   into the wrapper — the registration step needs to reference it
   independently of rendering.
2. Add the wrapper to that channel's registry object, same as adding a base
   template.
3. Add one `{ channel, name, parent, transform }` entry to that same file's
   exported `derivedTemplates` array — **the only registration step**; both
   the seeding manifest and `resolve()`'s transform lookup aggregate from
   every `transactional/*` module's `derivedTemplates` array automatically.

Skipping step 3 means the database path never learns this template exists at
all — behaves exactly like a plain code-only wrapper, `parent` chain and
all. `parent` and `transform` are declared together in the same array entry
by design, so there's no way to wire one without the other — no
half-registered state to worry about.

## Checklist before adding or editing a derived template

- [ ] Is the transform function exported and standalone, not inlined into
      the rendering wrapper?
- [ ] Is the `{ channel, name, parent, transform }` entry added to
      `derivedTemplates` — the only registration step, easy to forget since
      everything else about a derived template looks like a normal wrapper?
- [ ] When editing content, is the edit going to the right record — the
      ancestor (to change what propagates) or the template's own record (to
      break it independent)? Setting `hash` on a content-less record is a
      silent no-op, not a working "force refresh."
- [ ] If a fallback record needs to stop rendering, is it being deactivated
      (`active: false`, permanent, no self-heal) or deleted (temporary
      without a restart, self-heals on the next one) — deliberately, not by
      habit?
- [ ] If promoting a derived template to a base template, has the real
      MongoDB duplicate-key collision been accounted for — is the promotion
      path (update in place) actually being hit, not a fresh insert?

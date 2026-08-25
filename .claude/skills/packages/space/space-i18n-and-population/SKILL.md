---
name: space-i18n-and-population
description: langPreHandler/langGuard (URL-prefix language routing), populationGuard (segment/tenant resolution), and loadMessages (flat i18n content catalogs with population overrides) — the three mechanisms that decide which content variant a request gets. Use when adding a language, a population/tenant, or a new message key.
---

Covers the three request-time mechanisms that decide *which content variant*
a request gets: language, population/tenant, and the actual message
catalog. For CSP/CSRF guards, see `space-middleware-and-security` — a
different concern (security posture, not content selection). File:line
references point at `~/Documents/Development/ZanixLibraries/space` — read
the real code there before assuming this summary is still accurate.

## Golden rule (token savings)

- These three mechanisms compose in one direction — language and population
  are resolved first (as request context), `loadMessages` then reads that
  context. Don't re-derive `lang`/`population` inside a message-loading
  change; consume `ctx.params.lang`/`ctx.population` as given.
- Verify a message catalog's real shape with the file layout below, not by
  reading `load-messages.ts`'s full implementation.

## Language routing: `langPreHandler` + `langGuard`

```ts
import { bootstrapServers } from '@zanix/server'
import { langPreHandler } from '@zanix/space'

await bootstrapServers({
  ssr: { preHandler: langPreHandler({ availableLangs: ['en', 'es'], defaultLang: 'en' }) },
})
```

```tsx
import { defineMiddleware, langGuard } from '@zanix/space'
export default defineMiddleware([langGuard()])
```

`langPreHandler` is a `PreHandler` (runs *before* route matching, not a
guard). Resolution order for a request missing its `/{lang}/...` prefix:
persisted `X-Znx-Lang` cookie → `Accept-Language` → `defaultLang`. It then
301-redirects to the prefixed path (`/products` → `/en/products`, `/` →
`/en`), setting the cookie on the same response. It always skips
framework-internal routes (`/health`, `/ready`, `/assets/`, `/icons/`,
`/manifest.webmanifest`, `/sw.js`) — `ignorePrefixes` **extends**, never
replaces, that list. Pages live under a `routes/[lang]/...` convention.
There's no per-route opt-out — every route is uniformly prefixed.

**Why `langGuard` exists separately**: `langPreHandler` only refreshes the
cookie on an actual redirect. A request already correctly prefixed (e.g. via
a language-switcher link) has no way to refresh a stale cookie through the
`PreHandler` alone — `langGuard` runs after route matching, reads `:lang`
from the matched route param, and refreshes the cookie. Wiring
`langPreHandler` without `langGuard` leaves stale cookies uncorrected.

Default cookie name `X-Znx-Lang` (customizable via `langGuard({
cookieName })`, must match `langPreHandler`'s if customized) — same
`X-Znx-` prefix constraint `space-middleware-and-security`'s CSRF cookie
has (see `naming-and-structure-conventions` for the `cookiesGuard`
mechanism behind it).

**Requires `@zanix/server >= 3.2.0`** — below that version, multiple guards
setting the same header (`Set-Cookie`) on one route silently clobber each
other instead of merging, breaking `populationGuard`+`langGuard`
coexistence on the same route.

## Population resolution: `populationGuard`

```tsx
import { defineMiddleware, populationGuard } from '@zanix/space'
export default defineMiddleware([populationGuard()])

loader = (ctx) => ({ population: ctx.population })
component = ({ population }) => <p>Showing content for: {population ?? 'default'}</p>
```

Safe to register app-wide, since it's purely additive and never rejects.
Resolution order: route param → query string → persisted cookie, resolved
**server-side** (SSR-first, avoiding a client-side flash of the wrong
content variant) and exposed as `ctx.population` in `loader`. When the
resolved value differs from the cookie, it sets `Set-Cookie` on the
response.

Default cookie `X-Znx-Population`, same `X-Znx-` prefix constraint as
above. **Deliberately not `HttpOnly`, unlike the CSRF cookie** — client-side
code is expected to read it.

**Caution**: if a shared HTTP cache sits in front of the app, it needs
`Vary` on this cookie — an SSR response varying per-visitor cookie can't be
cached uniformly, and nothing in `@zanix/space` itself assumes a shared
cache exists. This guard only resolves *which* population; actual content
resolution for it is `loadMessages`'s job.

## Content resolution: `loadMessages`

```ts
// space.app.ts
export default defineSpaceApp({ name: 'storefront', messagesDir: './messages' })
```

```
messages/
  en/
    index.json                 # base catalog: { "home/title": "Welcome" }
    populations/
      zanix.json                # override: only the keys that differ from the base
  es/
    index.json
```

```tsx
import { loadMessages } from '@zanix/space'

loader = async (ctx: { params: { lang: string }; population?: string }) => ({
  messages: await loadMessages({ lang: ctx.params.lang, population: ctx.population }),
})
component = ({ messages }) => <h1>{messages['home/title']}</h1>
```

`messagesDir` accepts an array — same first-match-wins host-composition
convention as `routesDir`/`assetsDir`, resolved independently per base/
override file. `loadMessages({lang, population?})` returns `Messages` — an
opaque, flat `Record<string,string>`, never inspected/interpreted by this
function itself. Base + override are shallow-merged (`{...base,
...override}`), cached for the process lifetime keyed by `` `${lang}:${population
?? ''}` ``; concurrent calls for the same uncached key share one in-flight
resolution. **Cache is bypassed entirely under `znx space dev`** — live-edit,
no restart, same as `assetsDir`'s dev behavior.

**Real correctness constraint, not just a style rule**: catalogs must be
flat, never nested — a nested shape would silently lose sibling keys on any
merge collision. A missing override file resolves to the base catalog only
(normal, no warning). A **missing base file logs a warning** and resolves to
`{}` — it does not throw; language-level fallback/redirect is
`langPreHandler`'s job, not this function's. A **malformed file** (invalid
JSON, or not a flat object) logs an error and is skipped — base and override
are validated independently, so a broken override degrades to base-only
rather than failing the whole render.

`Messages` returns plain strings only, with no `react-intl`/formatting-library
coupling in this path. Cross-references:

- `@zanix/cli`'s `zanix space build` compiles `messagesDir`'s ICU strings
  into AST in place before formatting runs — catalogs may freely mix
  compiled/uncompiled values across keys.
- `@zanix/space-ui`'s `IntlProvider`/`useIntl`/`createFormatter` (React and
  Preact bindings, each independent — never `preact/compat`) wraps
  `@formatjs/intl`'s `createIntl()`, the only FormatJS dependency in the
  stack.
- `znx space dev` never runs the compiler — the dev-mode cache bypass plus
  `space-ui`'s formatter accepting either raw ICU or precompiled AST means
  nothing needs compiling in dev.

## Checklist before adding a language, population, or message key

- [ ] Are both `langPreHandler` **and** `langGuard` wired — not just the
      `PreHandler` alone, which leaves stale cookies uncorrected for
      already-prefixed requests?
- [ ] Is `@zanix/server >= 3.2.0` actually satisfied if both
      `langGuard`/`populationGuard` are registered on the same route?
- [ ] Does every new/changed catalog file stay flat — no nested objects that
      could silently lose sibling keys on merge?
- [ ] Does a shared HTTP cache in front of this app vary on the population
      cookie, if one exists?

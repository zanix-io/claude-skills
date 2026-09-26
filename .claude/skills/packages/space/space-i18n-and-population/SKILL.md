---
name: space-i18n-and-population
description: langPreHandler/langGuard (URL-prefix language routing), populationGuard (segment/tenant resolution), and loadMessages (flat i18n content catalogs with population overrides, splittable into per-feature segment files like iam.json/profile.json, plus defineSpaceApp({ messageSources }) for catalogs a library ships) — the three mechanisms that decide which content variant a request gets. Use when adding a language, a population/tenant, a new message key, splitting a growing catalog into segments, or shipping/overriding a package's default messages.
---

Covers the three request-time mechanisms that decide *which content variant*
a request gets: language, population/tenant, and the actual message
catalog. For CSP/CSRF guards, see `space-middleware-and-security` — a
different concern (security posture, not content selection). File:line
references point at `~/Documents/Development/ZanixLibraries/space` — read
the real code there before assuming this summary is still accurate.

Starting a NEW consumer project that needs this from day one: `zanix new
space|spacecraft --template population` (`populationGuard()` only, one
implicit locale, no `/[lang]/` prefix) or `--template population-lang`
(adds `langPreHandler`/`langGuard` and real `/[lang]/...` routing on top —
the full reference shape) scaffolds a working `src/space/middleware.ts` +
`messagesDir` wiring for exactly the mechanisms below, instead of hand-
wiring them from scratch. `--icons`/`--theme`/`--pages`/`--renderer` all
compose freely with either template value — see `cli-scaffold-assembly` for
the full preset/axis reference. An EXISTING project adding one of these
mechanisms for the first time still follows this skill directly.

## Golden rule (token savings)

- These three mechanisms compose in one direction — language and population
  are resolved first (as request context), `loadMessages` then reads that
  context. Don't re-derive `lang`/`population` inside a message-loading
  change; consume `ctx.params.lang`/`ctx.population` as given.
- Verify a message catalog's real shape with the file layout below, not by
  reading `load-messages.ts`'s full implementation.

## Language routing: `langPreHandler` + `langGuard`

```ts
// space.app.ts (or any module it imports — NOT mod.ts alone, see below)
import { definePreHandler, langPreHandler } from '@zanix/space'

definePreHandler(langPreHandler({ availableLangs: ['en', 'es'], defaultLang: 'en' }))
```

```ts
// mod.ts
import { getUserPreHandler } from '@zanix/space'
import { bootstrapRemoteApp } from '@zanix/app/runtime' // or bootstrapServers directly

await bootstrapRemoteApp(spaceApp, {
  server: { ssr: { preHandler: getUserPreHandler() } },
})
```

**Register via `definePreHandler`, not a literal `preHandler:` passed only to `mod.ts`'s own
bootstrap call.** `zanix space dev` never imports `mod.ts` at all — only `space.app.ts` — and boots
its own SSR server with a hardcoded, dev-only `preHandler` (Vite hot-client/asset handling),
composing a registered `getUserPreHandler()` result AFTER that. A `preHandler` declared only in
`mod.ts` is invisible under `dev`: `GET /` (an unprefixed URL) 404s instead of redirecting, while
working fine in production — this exact gap was confirmed and fixed (`@zanix/space`
`definePreHandler`/`getUserPreHandler`, `@zanix/cli`'s `dev/action.ts` composition) as of
2026-08-28. Same timing rule `defineMiddleware`'s guards already have: call it from something
`space.app.ts` imports (directly or transitively), never `mod.ts`-only.

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

**Cookie consent**: neither `X-Znx-Lang` nor `X-Znx-Population` (below) can
be gated behind a consent choice — no built-in option to skip the
`Set-Cookie`. Suggested classification for a consuming app's own
cookie-banner: "strictly necessary/functional" (the URL/query/route param
can also carry the value on every request; losing the cookie only loses
persistence across visits, never breaks the app) — see `docs/middleware.md`'s
own "Cookie consent" section in `@zanix/space` for the full reasoning.

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
    index.json                 # base catalog segment: { "home/title": "Welcome" }
    iam.json                    # another base segment — merged with index.json, see below
    profile.json
    populations/
      zanix.json                # override: only the keys that differ from the merged base
  es/
    index.json
    iam.json
    profile.json
```

**Requires `@zanix/space >= 1.12.0`** for segment merging — below that version only
`{lang}/index.json` is read as the base catalog; a segment file like `iam.json` is silently ignored
by an older version, not an error.

```tsx
import { loadMessages } from '@zanix/space'
import { IntlProvider, useIntl } from '@zanix/space-ui'

loader = async (ctx: { params: { lang: string }; population?: string }) => ({
  lang: ctx.params.lang,
  messages: await loadMessages({ lang: ctx.params.lang, population: ctx.population }),
})
// NEVER interpolate `messages[key]` directly as a JSX child — see `Messages`'s own note below for
// why (a compiled catalog value is precompiled AST, not a string). Always format through
// `IntlProvider`/`useIntl`, which accepts either shape and always returns a plain string.
component = ({ lang, messages }) => (
  <IntlProvider locale={lang} messages={messages}>
    <Home />
  </IntlProvider>
)
function Home() {
  const { formatMessage } = useIntl()
  return <h1>{formatMessage('home/title')}</h1>
}
```

`messagesDir` accepts an array so a host composes a base app's catalogs with
its own directory. Since `@zanix/space` 1.16.0 a file present in several
roots is merged **key by key**, the earlier root winning a shared key and a
key only a later root defines still resolving — so a host's file only needs
the messages it changes. Base segments and the population override each
merge this way independently. `loadMessages({lang, population?})` returns `Messages` — a
flat `Record<string, string | CompiledMessageNode[]>`, never inspected/
interpreted by this function itself (`CompiledMessageNode` mirrors
`@formatjs/icu-messageformat-parser`'s own AST node shape, redeclared
locally — `@zanix/space` never actually depends on FormatJS).

**The base catalog is every `.json` file found directly under `{lang}/`**
(never recursing into `populations/`, reserved for overrides), merged
filename-sorted — `index.json` is a fine single-file default, not a
hardcoded requirement. Split a growing catalog into feature-segmented files
instead (`iam.json`, `profile.json`, `chat.json`, ...) whenever one file
gets unwieldy — segments are expected to be namespaced/disjoint
(`'profile/name'`, `'chat/heading'`), so merge order only matters for the
unrecommended case of two segments sharing a key. The merged base, then the
population override, are shallow-merged (`{...segments, ...override}`),
cached for the process lifetime keyed by `` `${lang}:${population ?? ''}` ``;
concurrent calls for the same uncached key share one in-flight resolution.
**Cache is bypassed entirely under `znx space dev`** — live-edit, no
restart, same as `assetsDir`'s dev behavior. `zanix space build` needs no
extra configuration for a segmented catalog — its own compiler already
walks every `.json` file under `messagesDir` recursively, not just
`index.json`.

**Real correctness constraint, not just a style rule**: catalogs must be
flat, never nested — a nested shape would silently lose sibling keys on any
merge collision. A missing override file resolves to the merged base
catalog only (normal, no warning). Finding **no base segment file at all**
logs a warning and resolves to `{}` — it does not throw; language-level
fallback/redirect is `langPreHandler`'s job, not this function's. A
**malformed file** (invalid JSON, or not a flat object) logs an error and is
skipped — every segment and the override are validated independently, so
one broken segment degrades the merge to every other valid segment plus the
override, never discarding unrelated valid content.

### A library shipping its own catalogs: `messageSources`

A package has no directory an app could list in `messagesDir`, so it ships
its default messages as a `MessagesSource` (type exported from
`@zanix/space`, `src/modules/i18n/messages-types.ts`) and the app declares
it — **requires `@zanix/space >= 1.16.0`**:

```ts
import { iamMessages } from '@zanix/iam/ui/sdk/messages'
import { iamCssSource } from '@zanix/iam/ui/styles' // see "…and its CSS" below

export default defineSpaceApp({
  name: 'storefront',
  messagesDir: './messages', // optional; sources work without it
  messageSources: [iamMessages],
  cssSources: [iamCssSource],
})
```

`type MessagesSource = (lang, population?) => Messages | undefined |
Promise<...>`. `loadMessages` calls each source once with `population`
omitted (its base) and, when the request has a population, once more with
it (its override: only the keys that differ). `undefined` means "nothing for
this request" and is normal. The result is used as returned — `zanix space
build` compiles `messagesDir` only, so a source that wants precompiled
values returns the AST itself (`iamMessages` does). A source that throws or
returns a non-flat value is logged and skipped, never failing the request.

Precedence (`resolve()` in `src/modules/i18n/load-messages.ts`, tests in
`src/@tests/unit/i18n/load-messages.test.ts`), lowest to highest:

1. sources' base catalogs (earlier source wins a shared key),
2. the app's `messagesDir` base segments — **the app always wins over a
   package's default**, so an app rewords any shipped message by defining
   that key in its own catalog, in any file name,
3. the population override: the app's `populations/{population}.json`
   merged over the sources' population answers, the app's again winning a
   shared key. A source's population answer therefore beats the app's
   *base* value for that key.

Registration differs from `cssSources`: `defineSpaceApp({ messageSources })`
**replaces** the registered list (`setMessageSources`), it doesn't append —
a host that declares its own list must repeat a base app's sources.

Real precedent: `@zanix/iam`'s `iamMessages` (`ui/sdk/messages.ts`) answers
its compiled catalog for the base of a language it ships and `undefined` for
any other language or any population, leaving those entirely to the app's
own `messagesDir`. Its hosted pages declare the same source in `iam`'s own
`space.app.ts`.

**…and its CSS: `cssSources`.** The same library pattern for stylesheets:
`CssSource = { name, css: string | () => string | Promise<string>, media? }`
(`src/modules/render/css-sources.ts`), e.g. `@zanix/iam`'s `iamCssSource`
(`ui/styles.ts`). It has no language or population axis at all. Cascade
placement, naming and the dev/build materialization belong to
`space-styling-and-theming`'s "Responsive delivery" section.

No `react-intl`/formatting-library coupling in this resolution path itself — it
returns whatever is on disk (raw ICU string or precompiled AST) unformatted.
Cross-references:

- `@zanix/cli`'s `zanix space build` compiles `messagesDir`'s ICU strings
  into AST, writing the result to `{clientBuildDir}/messages/{rootIndex}/...`
  — NEVER back into `messagesDir` itself (that used to happen, silently
  corrupting a developer's own hand-authored ICU source on an ordinary local
  build; fixed as of 2026-08-28, same "compiled output lives in its own
  directory" contract `clientBuildDir` already has for the client bundle).
  Catalogs may freely mix compiled/uncompiled values across keys.
- `loadMessages()` reads from `{clientBuildDir}/messages/...` in production
  once `clientBuildDir` is configured (`getMessagesBuildDir()`) — `messagesDir`
  itself is only ever read live under `znx space dev`, which never runs the
  compiler at all: the dev-mode cache bypass plus `space-ui`'s formatter
  accepting either raw ICU or precompiled AST means nothing needs compiling
  in dev.
- `@zanix/space-ui`'s `IntlProvider`/`useIntl`/`createFormatter` (React and
  Preact bindings, each independent — never `preact/compat`) wraps
  `@formatjs/intl`'s `createIntl()`, the only FormatJS dependency in the
  stack.

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
- [ ] Is `preHandler` (e.g. `langPreHandler`) registered via `definePreHandler`
      from something `space.app.ts` imports — never only as a literal passed
      to `mod.ts`'s own bootstrap call, which `zanix space dev` can't see?
- [ ] Does every place a message renders go through `IntlProvider`/`useIntl`
      — never `messages[key]` interpolated directly (crashes once
      `zanix space build` compiles the catalog to AST)?
- [ ] Is `clientBuildDir` declared if this app wants `loadMessages()` to read
      compiled catalogs in production — without it, production falls back
      to reading `messagesDir` live (uncompiled ICU strings, never AST)?
- [ ] If splitting a base catalog into segment files (`iam.json`,
      `profile.json`, ...), is `@zanix/space >= 1.12.0` actually satisfied —
      an older version silently ignores every segment but `index.json`,
      not an error?
- [ ] Rewording a message a package ships through `messageSources` (e.g.
      `iamMessages`)? Define the key in this app's own `messagesDir`, never
      fork the package. If this app declares its own `messageSources` on
      top of a base app's, does it repeat the base's sources (the list
      replaces, it doesn't append)?

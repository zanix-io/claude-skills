---
name: space-middleware-and-security
description: The zero-config CSP/security-header defaults every page gets, the three-tier precedence (page > guard > framework default), static headers overrides, defineMiddleware/cspGuard/securityHeadersGuard, and csrfGuard — including a real bug where a misnamed cookie becomes invisible to it. Use when adding a guard, changing a page's security headers, or wiring CSRF protection.
---

Covers `@zanix/space`'s security-header defaults and its two dedicated
guards (CSP/security-headers, CSRF). For language/population routing guards
(`langGuard`/`populationGuard`), see `space-i18n-and-population` — a
different concern (content variant, not security posture). File:line
references point at `~/Documents/Development/ZanixLibraries/space` — read
the real code there before assuming this summary is still accurate.

## Golden rule (token savings)

- Trust the zero-config defaults for a page with no special security need —
  only reach for `static headers`/`cspGuard`/`securityHeadersGuard` when a
  page genuinely needs to diverge (an embedded payment iframe, a custom
  nonce policy).
- Check the three-tier precedence table once before assuming which value
  actually wins for a given field — don't guess from just "page vs. default."

## Zero-config defaults

Every page gets, automatically: `Content-Security-Policy: default-src
'self'; script-src 'self' 'nonce-<random per request>'`,
`X-Frame-Options: SAMEORIGIN`, `Referrer-Policy: strict-origin-when-cross-origin`,
`X-Content-Type-Options: nosniff`. The nonce is generated fresh per request
by `SpacePageController`, passed to React's `renderToReadableStream({
nonce })`, and matched in both the CSP header and the script tag's `nonce`
attribute.

## Overriding: `static headers`, app-wide default, and merge semantics

```tsx
export default class CheckoutPage extends SpacePageController {
  static headers = {
    csp: { 'default-src': ["'self'"], 'frame-src': ['https://payments.example.com'] },
    frameOptions: 'DENY' as const,
  }
  component = CheckoutView
}
```

```ts
export default defineSpaceApp({
  name: 'storefront',
  headers: { frameOptions: 'DENY', csp: { 'default-src': ["'self'"] } },
})
```

Fields: `csp`, `frameOptions`, `referrerPolicy`, `noSniff`,
`permissionsPolicy`, `strictTransportSecurity`, `crossOriginOpenerPolicy`,
`crossOriginEmbedderPolicy`, `crossOriginResourcePolicy`.

- `headers: false` disables everything (including CSP) for that page;
  `headers: { csp: false }` disables just CSP, keeping the rest of the
  defaults. `false` on any individual field wins over any registered guard —
  the header is completely absent, never empty, never the guard's value,
  never comma-joined.
- Page-level and app-wide `headers` merge **field-by-field** — a page
  overriding one field doesn't lose the rest. `csp` itself is the one
  exception: it merges as a **whole object**, not directive-by-directive — a
  page's own `csp` fully replaces the app-wide `csp`, it doesn't layer on
  top of it.
- Fields with no zero-config default (`permissionsPolicy`,
  `strictTransportSecurity`, and the three cross-origin-isolation fields)
  stay **absent** unless explicitly configured — there's no fallback value
  for these.
- Cross-origin isolation headers (`crossOriginOpenerPolicy`/
  `crossOriginEmbedderPolicy`/`crossOriginResourcePolicy`) are **off by
  default**, because strict values break OAuth/payment popups and
  third-party embeds unless the whole app is updated to account for them.

**Caution — a custom static `csp` loses automatic nonce coordination.** A
page setting its own `csp` object must either permit `'unsafe-inline'` or
build a custom nonce-based policy via `cspGuard`'s function form, or the
initial-state script won't survive a strict `script-src`. There's no
`<Helmet>`-style component API for headers, deliberately — headers must be
decided before streaming starts, so component-tree rendering can never
influence them.

## Guards: `defineMiddleware`, `cspGuard`, `securityHeadersGuard`, and precedence

```ts
defineMiddleware([
  cspGuard({ 'default-src': ["'self'"] }),
  securityHeadersGuard({ frameOptions: 'DENY' }),
])
```

`defineMiddleware(guards)` registers `MiddlewareGuard[]` for every SSR route
via `@zanix/server`'s `registerGlobalGuard` — scoped to `'ssr'` routes only,
process-wide, with no per-`Application` scoping.

**Precedence, field-by-field, three tiers**:
1. A page's own explicit `headers` (or the app-wide `defineSpaceApp({
   headers })` default, if the page sets none for that field).
2. A guard registered via `defineMiddleware`/`@Guard`
   (`cspGuard()`/`securityHeadersGuard()`).
3. The framework's zero-config default.

For non-CSP/header guards (rate limiting, custom auth checks), reuse
`@zanix/server`'s own `Guard` decorator directly — `@zanix/space` doesn't
wrap or reimplement it:

```ts
import { Guard } from '@zanix/server'
import { rateLimitGuard } from '@zanix/auth'
import { Page, SpacePageController } from '@zanix/space'

@Page('checkout')
@Guard(rateLimitGuard({ windowSeconds: 60, anonymousLimit: 20 }))
export default class CheckoutPage extends SpacePageController {
  component = CheckoutView
}
```

## CSRF protection: `csrfGuard`

```tsx
import { Guard } from '@zanix/server'
import { csrfGuard, Page, SpacePageController } from '@zanix/space'

@Page('checkout')
@Guard(csrfGuard())
export default class CheckoutPage extends SpacePageController {
  loader = (ctx) => ({ csrfToken: ctx.csrfToken })
  component = CheckoutView
  action = async (ctx) => { /* csrfGuard already validated by the time this runs */ }
}

function CheckoutView({ csrfToken }: { csrfToken?: string }) {
  return (
    <form method='post'>
      <input type='hidden' name='_csrf' value={csrfToken} />
    </form>
  )
}
```

**Not applied by default, deliberately** — an automatic default risks
silently breaking any existing `action` that doesn't yet render the token.
On `GET`, `csrfGuard()` issues/reuses a token in an `HttpOnly` cookie and
exposes it as `ctx.csrfToken` in `loader`; on other methods it rejects
unless the submitted token (`_csrf` form field, or `X-Znx-Csrf-Token` header
for fetch/XHR) matches the cookie.

**Real bug, worth knowing before naming any cookie in a Space app**: a
cookie named `znx-csrf` (lowercase, no `X-` prefix) was issued and echoed
correctly at the HTTP level but stayed **silently invisible** to
`csrfGuard` — see `naming-and-structure-conventions` for why
(`cookiesGuard`'s prefix filter). `csrfGuard`'s cookie name **must start
with `X-Znx-`**; default is `X-Znx-Csrf`. The same constraint applies to
`populationGuard`'s cookie (`space-i18n-and-population`).

`@zanix/auth`'s session cookies default to `SameSite=Strict` already, which
mitigates classic CSRF — `csrfGuard` is defense-in-depth on top of that, or
a substitute for apps not using `@zanix/auth` at all.

## Checklist before touching page security headers or CSRF

- [ ] Does a page-level `headers` override still coordinate with the
      automatic nonce if it sets a custom `csp`?
- [ ] Is the precedence tier (page/app-wide > guard > default) the actual
      reason a field resolves the way it does — not assumed from reading
      only one of the three?
- [ ] If enabling `csrfGuard`, does every existing `action` on that route
      already render `_csrf`/send `X-Znx-Csrf-Token` — not just the newly
      added one?
- [ ] Does any new cookie this app sets start with `X-Znx-` — otherwise
      `cookiesGuard` silently hides it from every guard that reads
      `ctx.cookies`?

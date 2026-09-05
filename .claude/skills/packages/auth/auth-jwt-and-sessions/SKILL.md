---
name: auth-jwt-and-sessions
description: Core registration (the ./core subpath's zero-config provider wiring), JWT create/verify/decode with JWK key rotation, session token generation/refresh/revoke, configurable access/refresh-token expirations, the accessToken-vs-token distinction, session response headers/cookies, the block list, the real refresh-token rotation mechanism — including a footgun where blocklisting can silently write without ever checking — the rotation grace window that tolerates legitimate near-simultaneous refresh requests, the freshness-window rotation cadence `pageSessionGuard` actually runs on (`deriveSessionToken`), and minting a real access token from an existing session without touching its refresh token (`mintAccessToken`). Use when touching JWT/session/token lifecycle.
---

Covers JWT handling and session lifecycle — the foundation `auth-oauth2`,
`auth-otp-and-totp`, and `auth-service-credential` all build sessions on top
of. File:line references point at
`~/Documents/Development/ZanixLibraries/auth` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- Trust `this.providers.get('auth').session`'s bound methods
  (`.generateTokens()`/`.refreshTokens()`/`.revokeToken()`) for anything
  inside a Zanix-managed class — reach for the standalone functions
  (`generateSessionTokens()`, etc.) only outside one.
- `options.cache` alone (with Redis configured) is sufficient for full
  refresh-token reuse detection — `options.kvDb` is an optional non-Redis
  fallback, never a second requirement. Don't assume both are needed before
  checking the rotation section below.
- Three functions verify an existing refresh token, each for a different
  job — don't reach for the wrong one: `refreshSessionTokensBase` (full
  pair, always rotates), `deriveSessionTokenBase` (claims only, never signs
  a real access token, rotates the refresh token only when it's stale —
  what `pageSessionGuard` runs on), `mintAccessTokenBase` (a real access
  token, but NEVER rotates/blocklists the refresh token — for a caller that
  needs a credential to hand outward, not to manage session lifecycle).

## Core registration: the zero-config wiring

```ts
import 'jsr:@zanix/auth/core'
```

Importing the default `jsr:@zanix/auth` entrypoint only exposes classes/
functions/types — it registers **nothing** with the framework by itself.
Importing the `./core` subpath once, anywhere it runs at startup, is what
registers the default `ZanixAuthProvider` under the `'auth'` core-provider
key, the default session-headers interceptor, and — when
`GOOGLE_OAUTH2_CLIENT_ID` is set — a default `GoogleOAuth2Connector`
(`auth-oauth2`). The same zero-config pattern applies to GitHub: when
`GITHUB_OAUTH2_CLIENT_ID` is set, a default `GitHubOAuth2Connector` is
registered too. An app bootstrapping via `@zanix/core`'s `Zanix.start()`/
`Zanix.startWorker()` already does this import. Once registered,
`this.providers.get('auth')` exposes `.google`, `.github`, `.otp`, `.totp`,
and `.session` — each bound to the current request context, so callers
never thread `ctx`/cache/connectors through the lower-level functions by
hand.

## JWT: create/verify/decode, HMAC and RSA

```
createJWT(payload, secret, opts?)   // create
verifyJWT(token, secret, opts?)     // verify
decodeJWT(token)                    // decode header/payload without verifying signature
getSecretByToken(token, type?)      // resolve the signing/verification secret for a given token
```

Supports both HMAC keys (`JWT_KEY`, `user` tokens) and RSA keys (`JWK_PRI`/
`JWK_PUB`, `api` tokens). JWK key rotation uses versioned keys (`_V1`,
`_V2`, …), controlled by `JWK_ROTATION_CYCLE` (a human-readable duration
like `"30m"`, or a numeric value in seconds) — rotation only occurs when
multiple versioned keys exist; with only one key, rotation is disabled.

## Reading the current request's token: `accessToken` vs. `token`

```ts
class ProxyInteractor extends ZanixInteractor {
  @AuthTokenValidation()
  async handle() {
    const upstream = await fetch('https://internal-service/resource', {
      headers: { Authorization: `Bearer ${this.context.session.accessToken}` },
    })
    return upstream.json()
  }
}
```

`ctx.session.accessToken` is **this request's own verified access token** —
the exact value from its `Authorization`/`X-Znx-Authorization` header, set
once `jwtValidationGuard`/`AuthTokenValidation` succeeds. **Never the same
thing as `token`** — the session's separate `token` field only exists once a
session is created/refreshed (`generateTokens()`, `refreshTokens()`, or any
OTP/TOTP/OAuth2 `authenticate()`), and it's the long-lived refresh token
`sessionHeadersInterceptor` stores in the `X-Znx-App-Token` cookie. A
request that only went through `jwtValidationGuard` (no login/refresh in the
same request) has `accessToken` but no `token`.

## Session tokens: generate, refresh, revoke

```
generateSessionTokens()   // both access + refresh tokens (login-time, mints fresh)
createAccessToken()
createRefreshToken()
deriveSessionToken()       // claims from an EXISTING refresh token, rotates only when stale
mintAccessToken()          // a real access token from an EXISTING refresh token, never rotates it
revokeAppTokens()          // revoke every token for an app
revokeSessionToken()       // revoke one session
```

`generateSessionTokens()`'s `AuthSessionOptions` accepts optional `accessExpiration`/
`refreshExpiration` (defaulting to `'1h'`/`'1y'` — `DEFAULT_ACCESS_EXPIRATION`/
`DEFAULT_REFRESH_EXPIRATION` in `utils/constants.ts`) instead of the fixed literals earlier
versions hardcoded. Whatever is chosen travels forward automatically into every later
refresh/derive/mint call for that session, since the whole `AuthSessionOptions` object is embedded
in the refresh token's own payload (`payload.access`). `refreshExpiration` must be at least
`MIN_REFRESH_TO_ACCESS_RATIO` (`3`) times `accessExpiration` — `generateSessionTokens()` throws
`InternalError('AUTH_SESSION_INVALID_EXPIRATION')` otherwise; this margin exists so a session that
goes quiet right around its rotation cadence still has real room left before the refresh token's
own absolute expiry (see the freshness-window section below for why that cadence exists at all).
Note `refreshExpiration`'s own type (`'1w'|'1mo'|'6mo'|'1y'`) already clears this margin against any
allowed `accessExpiration` (capped at `'1h'`) for a type-honest TypeScript caller — the runtime
check mainly guards a caller bypassing the type or calling from plain JavaScript.

## Refresh-token rotation: single-use, blocklist-backed reuse detection

`refreshSessionTokens(ctx, token?, options)` verifies the given refresh
token, generates a fresh set, and — **only when `options.cache` is
provided** — blocklists the one just consumed (single-use rotation), so a
later attempt to reuse the same token (a leaked log line, a stolen cookie
replayed after the legitimate client already rotated) is rejected instead
of silently accepted.

Both the reuse **check** and the blocklist **write** gate on `options.cache`
alone — `options.kvDb` is optional for both, consulted only as a fallback
when `REDIS_URI` isn't set and the in-memory local cache has no entry (the
Redis path never touches it). Passing `cache` alone (with Redis configured)
is a fully supported, complete configuration — it gets both the write and
the check. Omitting `cache` entirely disables rotation altogether — a
deliberate trade-off for callers without blocklist infra, not a silent gap,
but it means a captured refresh token stays valid until its own `exp`.

### Rotation grace window: tolerating two legitimate near-simultaneous requests

Single-use rotation is strict by nature — the first request to consume a
token wins, and a second request presenting that same token a moment later
looks identical to a stolen-token replay. A refresh token check-then-write
cycle isn't atomic across two genuinely legitimate concurrent callers
(a browser prefetching a link on hover and then navigating it, a double
click, two tabs on the same session) — one succeeds normally, and the other
would otherwise land on an already-blocklisted token and get rejected, even
though it presented a token that was still valid the moment it was sent.

`refreshSessionTokensBase` (`utils/sessions/refresh.ts`) closes this without
weakening reuse detection past a short window: before rejecting a
blocklisted token outright, it looks up `getRotationGraceTokens(jti, cache,
kvDb)` (`utils/sessions/block-list.ts`) — the pair that same token was
already rotated into, written by `setRotationGraceTokens` **before** the
blocklist entry itself, closing the gap where a concurrent request could
otherwise observe "blocklisted" with no grace entry to answer it yet. A hit
returns that same pair with a normal response, `ctx.locals.session`
populated exactly as a freshly minted rotation would populate it; a miss
(past the grace window, or a genuinely different/stolen token) still
rejects. Window length: `ROTATION_GRACE_WINDOW` (`ROTATION_GRACE_WINDOW_ENV`
in `utils/constants.ts`), default `5s`. This only changes behavior for a
token presented again within that window after its own rotation — it adds
no tolerance for a token replayed well after the legitimate client moved on.

## Rotation cadence for page guards: `deriveSessionToken`, not every page load

`pageSessionGuard` does NOT call `refreshSessionTokens` — it calls `deriveSessionToken`
(`utils/sessions/derive.ts`), built specifically because a `@zanix/space` page never needs a real
signed access token (only the claims it would carry), and because rotating the refresh token on
literally every page load — the old behavior — churns the blocklist/rotation-grace cache for no
security benefit once a session is browsed at any normal pace.

`deriveSessionTokenBase` decides whether to rotate by comparing the presented refresh token's `iat`
against its own embedded `accessExpiration` (default `'1h'`): younger than that → reused as-is, no
mint, no cache write, no new cookie; at least that old → a real rotation, delegated whole to
`refreshSessionTokensBase` (same single-use blocklisting/grace window, unchanged). In practice a
session rotates roughly once per `accessExpiration` window per active browsing session, not once
per click.

`options.rotateRefresh?: boolean` overrides that automatic decision: `true` forces a rotation even
for a fresh token (e.g. right before a sensitive action, to invalidate any other copy of the
current refresh token in circulation); `false` forces reuse even for a stale one — deliberately, for
a call site that has NO business deciding whether a session renews (see `mintAccessToken` below for
exactly that call site). Forcing `false` on the only call site presenting a given refresh token
means its absolute lifetime is never renewed by that call — a deliberate consequence of opting out,
not a bug, since renewal is meant to happen via whatever DOES gate the surrounding page/route.

## Minting a real access token from an existing session: `mintAccessToken`

For a caller that needs to act ON BEHALF OF the current session — most commonly an SSR-side handler
relaying a request to an external Zanix service as `Authorization: Bearer` — but has no business
deciding whether the session's refresh token should rotate (that decision belongs exclusively to
whatever already gates the page/route this call happens inside, e.g. `pageSessionGuard`).
`mintAccessTokenBase`/`mintAccessToken` (`utils/sessions/mint-access-token.ts`) verify the presented
refresh token and sign a real access token from its embedded `payload.access` — never touching the
refresh token itself: no replacement minted, nothing blocklisted, nothing written to the
rotation-grace cache. A token already sitting in another rotation's grace window (rotated by a
concurrent request moments earlier) hands back that already-real access token as-is instead of
minting a redundant one — the caller never needs to know or branch on which case happened.

This is the service-to-service `createServiceAuthClient`'s conceptual sibling for the human-session
case — and, like it, `@zanix/auth` deliberately stops at producing the credential/header. It never
executes the outbound request itself (no `fetch`/`RestClient` call to the external target lives in
this package) — that stays the caller's own HTTP client, same boundary `createServiceAuthClient`
already draws.

```ts
// inside an SSR handler whose OWN guard already resolved a session
const { accessToken } = await mintAccessToken(ctx, undefined, { cache })
const response = await fetch('https://api-externa.zanix/recurso', {
  headers: { Authorization: `Bearer ${accessToken}` },
})
```

Don't reach for `deriveSessionToken` here even with `rotateRefresh: false` — it's built to AVOID
producing a real signed access token (`pageSessionGuard`'s own need), the opposite of what this call
site wants.

## Guard-stage rotation recovery

`@zanix/server`'s guard pipeline skips its registered response interceptors
whenever a GUARD throws (unlike a handler-body throw, whose own recovery
path still runs interceptors) — so a guard that calls `refreshSessionTokens`/
`deriveSessionToken` and then a later check (a permission check, most commonly) throws never gets
`sessionHeadersInterceptor`'s chance to deliver the replacement cookie. The
client is left holding a cookie the rotation already blocklisted, with the
replacement computed but never sent. Only actually happens on a request where a rotation ran in
the first place — `deriveSessionToken`'s own freshness-window skip leaves nothing to recover, so
this gap fires far less often under `pageSessionGuard` than it used to.

`attachRotatedSessionToError(error, ctx)` / `recoverRotatedSessionCookie()`
close that gap, for exactly this guard shape — rotation followed by a later
check in the same guard, e.g. `refreshSessionTokens` + `permissionsPipe`:

```ts
import { attachRotatedSessionToError, refreshSessionTokens } from '@zanix/auth'

export function requireSession(roles: string[]): MiddlewareGuard {
  return async (ctx) => {
    await refreshSessionTokens(ctx, undefined, { cache })
    try {
      await requirePermissions(ctx)
    } catch (error) {
      throw attachRotatedSessionToError(error, ctx)
    }
    return {}
  }
}
```

`pageSessionGuard` is this exact composition, ready-made, built on `deriveSessionToken` instead of
`refreshSessionTokens` directly — reach for it before hand-rolling the guard above.

`attachRotatedSessionToError` marks `error` with a non-enumerable own
property (invisible to `serializeError`/`console.error`/`JSON.stringify`,
same discretion `@zanix/server`'s own `attachRequestToError` applies) and
returns it unchanged — a no-op when `ctx.locals.session` carries no `token`
(nothing to recover). Wire `recoverRotatedSessionCookie()` as
`server.ssr.onError`, typically composed with other error handlers via
`@zanix/space`'s `globalErrorHandler()` (see `space-routing-and-rendering`):
it declines (`undefined`) for any error the attach call never touched, and
rebuilds the response via this package's own `getSessionHeaders`/
`addHeadersToResponse` — the same functions `sessionHeadersInterceptor`
itself uses — so the recovered cookie's attributes stay identical to every
other path that sets one.

Only relevant to a guard that rotates a session and can still throw
afterward in the same chain — a guard that only ever rotates and returns has
nothing to recover, since `sessionHeadersInterceptor` delivers the cookie
normally on any successful response.

## Block list

```
addTokenToBlockList(token, cache, kvDb?)   // add by jti, extracted internally
checkTokenBlockList(jti, cache, kvDb)      // check whether a token ID is blocked
```

## Session response headers and cookies

When a valid session is present: `x-znx-<type>-session-status`,
`x-znx-<type>-id` (when the token has a `sub` claim). When
`X-Znx-Cookies-Accepted: true` is present, session cookies are also set via
`Set-Cookie`: `X-Znx-App-Token` (the refresh token, `Max-Age` derived from
its own expiration — e.g. ~1 year), `X-Znx-<type>-Session-Status`,
`X-Znx-<type>-Id`, `X-Znx-Cookies-Accepted` (the latter three `Max-Age`
derived from the access token's expiration, capped at 1h). All cookies are
`HttpOnly`/`Secure`/`SameSite=Strict`. Names come from
`AUTH_HEADERS`/`SESSION_HEADERS`/`GENERAL_HEADERS`, exported from
`@zanix/server`, not `@zanix/auth` itself.

Same `X-Znx-` cookie-prefix constraint as every other Zanix cookie — see
`naming-and-structure-conventions` for the mechanism
(`cookiesGuard` silently hides anything outside that prefix from
`ctx.cookies`). A cookie set anywhere in this package outside that prefix is
silently invisible to every guard reading `ctx.cookies`.

## Checklist before touching JWT/session code

- [ ] Is the session accessed via `this.providers.get('auth').session`
      inside a Zanix-managed class, rather than threading `ctx`/cache
      manually?
- [ ] For refresh rotation, is `options.cache` actually provided — that
      alone is enough for full reuse detection; `options.kvDb` is an
      optional non-Redis fallback, not a second requirement?
- [ ] If a guarded route sees legitimate near-simultaneous requests (client
      prefetch + navigation, a double click, two tabs), is the rotation
      grace window (`ROTATION_GRACE_WINDOW`, default `5s`) actually what's
      relied on — not a client-side workaround (disabling prefetch) that a
      grace window already makes unnecessary?
- [ ] Is `accessToken` vs. `token` used correctly — never assuming a
      request that only validated a JWT also carries a refresh `token`?
- [ ] Picking a function to verify an existing refresh token: full pair +
      always rotates (`refreshSessionTokensBase`), claims only + rotates
      when stale (`deriveSessionTokenBase`), or a real access token that
      never touches the refresh token at all (`mintAccessTokenBase`) — not
      just reaching for whichever one is already imported nearby?
- [ ] If configuring `accessExpiration`/`refreshExpiration`, does
      `refreshExpiration` actually clear `MIN_REFRESH_TO_ACCESS_RATIO` (`3`)
      times `accessExpiration` — not just "greater than"?
- [ ] Does any new cookie this package sets keep the `X-Znx-` prefix?
- [ ] Does a guard rotate a session AND throw afterward in the same chain
      (e.g. rotation followed by a permission check)? If so, is the
      downstream throw wrapped in `attachRotatedSessionToError`, with
      `recoverRotatedSessionCookie()` wired into the consumer's own
      `onError` chain — not just assumed to work because a successful
      response would have delivered the cookie fine?

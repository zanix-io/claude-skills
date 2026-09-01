---
name: auth-jwt-and-sessions
description: Core registration (the ./core subpath's zero-config provider wiring), JWT create/verify/decode with JWK key rotation, session token generation/refresh/revoke, the accessToken-vs-token distinction, session response headers/cookies, the block list, and the real refresh-token rotation mechanism — including a footgun where blocklisting can silently write without ever checking. Use when touching JWT/session/token lifecycle.
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
- Check the refresh-rotation gating table below before assuming
  `options.cache` alone gives full reuse protection — it doesn't.

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
generateSessionTokens()   // both access + refresh tokens
createAccessToken()
createRefreshToken()
revokeAppTokens()          // revoke every token for an app
revokeSessionToken()       // revoke one session
```

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

## Guard-stage rotation recovery

`@zanix/server`'s guard pipeline skips its registered response interceptors
whenever a GUARD throws (unlike a handler-body throw, whose own recovery
path still runs interceptors) — so a guard that calls `refreshSessionTokens`
and then a later check (a permission check, most commonly) throws never gets
`sessionHeadersInterceptor`'s chance to deliver the replacement cookie. The
client is left holding a cookie the rotation already blocklisted, with the
replacement computed but never sent.

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
- [ ] Is `accessToken` vs. `token` used correctly — never assuming a
      request that only validated a JWT also carries a refresh `token`?
- [ ] Does any new cookie this package sets keep the `X-Znx-` prefix?
- [ ] Does a guard rotate a session AND throw afterward in the same chain
      (e.g. rotation followed by a permission check)? If so, is the
      downstream throw wrapped in `attachRotatedSessionToError`, with
      `recoverRotatedSessionCookie()` wired into the consumer's own
      `onError` chain — not just assumed to work because a successful
      response would have delivered the cookie fine?

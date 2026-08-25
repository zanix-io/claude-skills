---
name: auth-permissions-and-rate-limiting
description: AuthTokenValidation vs. RequirePermissions/permissionsPipe (JWT-signature-time check vs. session.scope check), the OR-not-AND semantics of multiple required permissions, the '*' wildcard, and rateLimitGuard's plan-based limits with the app-scoping option. Use when gating a route by permission/scope, or configuring rate limits.
---

Covers authorization (permissions/scopes) and rate limiting — the two
per-request checks layered on top of a valid session. For how the session
itself is created/verified, see `auth-jwt-and-sessions`. File:line
references point at `~/Documents/Development/ZanixLibraries/auth` — read the
real code there before assuming this summary is still accurate.

## Golden rule (token savings)

- Default to `AuthTokenValidation({ permissions })` for a normal route — it
  authenticates and authorizes in one decorator. Reach for
  `RequirePermissions`/`permissionsPipe` only when authentication already
  happened earlier and a specific method needs a narrower check on top.
- Multiple required permissions are OR, not AND — check this before writing
  an authorization test that assumes all of them are required.

## `AuthTokenValidation` vs. `RequirePermissions`/`permissionsPipe`

The `permissions` set when creating session tokens
(`generateTokens()`/`authenticate()`, any flow) are stored as the JWT's
`aud` (audience) claim — everything below checks against that same claim,
just at different points:

- **`AuthTokenValidation({ permissions })`** checks `permissions` against
  the token's `aud` claim as part of JWT verification itself, before a
  session even exists. This is the common case — one decorator both
  authenticates the request and authorizes it.
- **`RequirePermissions`/`permissionsPipe`** checks the same `permissions`
  against `ctx.session.scope` (the `aud` claim copied onto the session once
  it exists) — it never touches the JWT signature. Use it only when
  authentication already happened earlier in the pipeline and one specific
  method needs an additional, narrower permission check.

```ts
class SecureInteractor extends ZanixInteractor {
  @AuthTokenValidation({ permissions: ['admin', 'write:user'] }) // any ONE of these is enough
  async handle() { /* ... */ }
}
```

**OR, not AND**: access is granted if **at least one** required permission
matches — `permissions: ['admin', 'write:user']` means "admin OR
write:user," never "both." A session whose `aud`/scope includes `'*'` is
granted access to any permission check.

## Rate limiting: `rateLimitGuard`

```env
RATE_LIMIT_WINDOW_SECONDS=60
RATE_LIMIT_PLANS=0:100;1:1000;2:3000
```

`session.rateLimit` selects which plan applies (e.g. `0` → 100
requests/window, `1` → 1000). If no plan index matches, the value directly
sets the allowed requests per window instead of indexing into
`RATE_LIMIT_PLANS`. Both `rateLimitGuard` and `jwtValidationGuard` accept an
`app` option to scope the rate-limit cache key per app, so multiple apps
sharing the same session/user ID don't collide on the same counter.

Response headers, when applicable: `X-Znx-RateLimit-Limit`,
`X-Znx-RateLimit-Remaining`, `X-Znx-RateLimit-Reset`, `Retry-After` — these
are the `RATE_LIMIT_HEADERS` constant, exported from `@zanix/server`, not
`@zanix/auth`.

`@zanix/auth`'s own service-credential path
(`auth-service-credential`) sets `session.rateLimit` via
`SERVICE_RATE_LIMIT_<serviceId>` — `rateLimitGuard` reads it off the JWT
identically for `api` and `user` sessions, with no type-specific branch.

### `anonymousLimit` and `trustProxyHeader`

`rateLimitGuard` accepts requests with no session at all by default —
`anonymousLimit` (default `100`) is the requests-per-window cap applied to
those anonymous callers, keyed off their IP. Because that key comes from a
client-controlled header unless the server sits behind a trusted proxy,
enabling anonymous access forces the same explicit decision
`auth-network-security` documents for `IpAllowlistGuard`: pass
`trustProxyHeader` — `true` to trust the proxy header, or `false` to key off
the raw connection IP instead. Leaving it `undefined` while `anonymousLimit`
is truthy throws an `InternalError` at guard-construction time (not on the
first request) — set `anonymousLimit: false` to disable anonymous access
outright if no such decision applies here.

## Checklist before gating a new route

- [ ] Is `AuthTokenValidation` used for the common case — reaching for
      `RequirePermissions`/`permissionsPipe` only when a genuinely separate,
      narrower check is needed on top of authentication that already
      happened?
- [ ] Does a multi-permission requirement actually mean "any of these" —
      not accidentally relying on all of them being present?
- [ ] Is `app` passed to `rateLimitGuard`/`jwtValidationGuard` when multiple
      apps could share the same session/user ID, to avoid a shared rate-limit
      counter?

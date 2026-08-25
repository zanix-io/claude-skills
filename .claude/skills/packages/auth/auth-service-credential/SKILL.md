---
name: auth-service-credential
description: Machine-to-machine authentication via createServiceAssertion/exchangeServiceCredential — service registration by JWK_PUB_<serviceId>, the PKCS#8-only key format, operator-configured (never caller-requested) permissions/rate limits, and the no-downtime key-rotation procedure via the _<keyId> suffix form. Use when a service needs to call another service's API with no shared secret.
---

Covers `@zanix/auth`'s service-credential exchange — a calling service
proves its identity with a short-lived, self-signed assertion and exchanges
it for a real `type: 'api'` access token. No shared secret, no human-shaped
session borrowed as a stopgap. File:line references point at
`~/Documents/Development/ZanixLibraries/auth` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- Omit `privateKey` and let it resolve `JWK_PRI_<serviceId>` automatically —
  only pass it explicitly when reading the key from somewhere other than an
  env var (a secrets manager, a config file).
- Reuse `@zanix/core`'s ready-made `/admin/service-token` handler instead of
  re-implementing the exchange endpoint, if the app is already
  `@zanix/core`-based.
- Calling the same target repeatedly? Use `createServiceAuthClient` instead
  of hand-rolling the sign → exchange → cache flow below — it already
  caches per target.

## Basic usage

```ts
// Calling service
import { createServiceAssertion } from 'jsr:@zanix/auth'
const assertion = await createServiceAssertion({ serviceId: 'zanix-admin' })
```

```ts
// Receiving service
import { exchangeServiceCredential } from 'jsr:@zanix/auth'
const credential = await exchangeServiceCredential(assertion)
// { accessToken, expiresIn, serviceId }

fetch(url, { headers: { 'X-Znx-Authorization': `Bearer ${credential.accessToken}` } })
```

`createServiceAssertion`'s `privateKey` is the calling service's own
keypair — never `@zanix/auth`'s own `JWK_PRI`. Omitted, it resolves
`JWK_PRI_<serviceId>` (or `JWK_PRI_<serviceId>_<keyId>` with `keyId` set)
automatically — the exact mirror of how the receiving side resolves
`JWK_PUB_<serviceId>`. `resolveServiceAssertionPrivateKey(serviceId, keyId)`
is exported to check resolvability upfront, e.g. to fail fast at boot before
any real signing attempt.

**Key-format gotchas, easy to get wrong**: the value is **base64-encoded**
(same convention as `JWK_PRI`/`JWK_PUB` elsewhere) — generate with
`generateRSAKeys()` (`@zanix/helpers`) and store `btoa(privateKey)`, not the
raw multi-line PEM (a raw PEM's `-`/newlines aren't valid base64 and fail to
decode). **Only PKCS#8 private keys are supported** (`-----BEGIN PRIVATE
KEY-----`) — the underlying `crypto.subtle.importKey` call always imports
with `format: 'pkcs8'`. A PKCS#1 key (`-----BEGIN RSA PRIVATE KEY-----`,
e.g. from `openssl genrsa`) fails to import; convert with `openssl pkcs8
-topk8 -nocrypt -in old.pem -out new.pem`. The matching public key must be
SPKI (`-----BEGIN PUBLIC KEY-----`), what `generateRSAKeys()`/`openssl rsa
-pubout` produce by default.

## `createServiceAuthClient` — the cached client for repeated calls

```ts
import { createServiceAuthClient } from 'jsr:@zanix/auth'

// privateKey omitted -> resolves JWK_PRI_zanix-admin automatically
const auth = createServiceAuthClient({ serviceId: 'zanix-admin' })

const headers = await auth('billing', 'http://billing.internal:8000/admin/service-token')
// { 'X-Znx-Authorization': 'Bearer <token>' }
await fetch('http://billing.internal:8000/admin/some-endpoint', { headers })
```

Wraps the exact two-call flow above (`createServiceAssertion` then the
exchange) plus a per-`targetServiceId` token cache, so a caller making
repeated calls to the same target doesn't have to hand-roll its own cache
or re-exchange on every call. Signs and exchanges a fresh assertion only on
a cache miss — nothing cached yet for that target, or the cached token is
within 5s of expiring (`EXPIRY_SAFETY_MARGIN_MS`). Takes the target's
`serviceId`/exchange URL per call rather than a fixed one, so one client
instance can authenticate against multiple targets, each cached
independently. `ServiceAuthClientOptions` mirrors `createServiceAssertion`'s
own options (`serviceId`/`privateKey`/`keyId`/`assertionExpiration`), plus
an optional `httpClient` to present a client certificate on the exchange
call itself.

The two-call flow above is still the right tool for a one-shot exchange, or
when the caller needs to own caching itself (e.g. sharing one cache across
processes). `ZanixAdminHub.start({ auth })` (`admin-service-authentication`)
builds on this same function internally for its own outbound calls, rather
than a parallel implementation.

## Registering a trusted service

```env
JWK_PUB_zanix-admin=<base64-encoded PEM public key>
```

No dynamic self-registration — a service only gets a token if the receiving
side explicitly registered its public key. `serviceId` doubles as the
assertion's `iss` and `sub`, so a valid signature alone can never be
presented as a different service's identity — `exchangeServiceCredential`
requires `sub === iss`.

For a real overlap window during key rotation, register a `_<keyId>`-suffixed
form instead of the bare single-key form:

```env
JWK_PUB_zanix-admin_key1=<current public key>
JWK_PUB_zanix-admin_key2=<incoming public key>
```

```ts
const assertion = await createServiceAssertion({
  serviceId: 'zanix-admin',
  keyId: Deno.env.get('SERVICE_KEY_ID')!,
})
```

`keyId` only selects *which* registered key verifies the assertion — it
never affects identity (`iss`/`sub` stay `serviceId`) or the permissions/
rate limit granted (always keyed by `serviceId` alone). A key registered
under one `serviceId` can never authenticate as a different one — the
lookup is always scoped by `serviceId` first.

## Permissions and rate limiting — operator-configured, never caller-requested

```env
SERVICE_PERMISSIONS_zanix-admin=ADMIN_ROLE,ADMIN_TRIGGERS_ROLE
SERVICE_RATE_LIMIT_zanix-admin=500
```

Granted permissions and rate limits come **only** from these
operator-configured env vars — never from anything the calling service's
own assertion requests. `SERVICE_RATE_LIMIT_<serviceId>` falls back to
`createAppToken`'s default (`100`) if unset, same as a `type: 'user'`
session's default. `rateLimitGuard` reads `session.rateLimit` identically
for `api` and `user` sessions (`auth-permissions-and-rate-limiting`).

## Mounting the exchange endpoint

```ts
@Post('service-token', { Body: ServiceExchangeRTO })
public exchange(ctx: HandlerContext<{ body: ServiceExchangeRTO }>) {
  return exchangeServiceCredential(ctx.payload.body.assertion)
}
```

**No role gate belongs on this route** — the caller has no session yet at
the point it's calling this endpoint; trust is established entirely by
`exchangeServiceCredential`'s own verification against
`JWK_PUB_<serviceId>`. `@zanix/core` ships a ready-made version of exactly
this at `/admin/service-token` — reuse it instead of re-implementing this
handler in a `@zanix/core`-based app.

## Rotating a service's key — no downtime, no deploy ordering

Only possible with the `_<keyId>`-suffixed form (the bare single-key form
has no overlap window):

1. Register the new key pair alongside the still-active one on both sides
   (`JWK_PUB_<serviceId>_key2` next to `_key1` on the receiving side,
   `JWK_PRI_<serviceId>_key2` next to `_key1` on the calling side). Both
   verify/can sign at this point; nothing needs to switch yet.
2. Switch the calling service's own `keyId` config and restart/redeploy —
   no code change. An assertion already in flight, signed with the old key
   before the restart, keeps verifying throughout the rollout; there's no
   requirement that every instance switch atomically.
3. Wait past the max assertion expiration (`createServiceAssertion`'s
   `expiration`, `2m` by default) before assuming no old-key assertion is
   still in flight.
4. Remove the old key (`JWK_PUB_<serviceId>_key1`). Assertions signed with
   it now fail; the new key is unaffected.

Rotating the calling service's assertion-signing key has no effect on
already-minted access tokens — those are signed with `@zanix/auth`'s own
`JWK_PRI` (a separate mechanism, see `auth-jwt-and-sessions`'s key
rotation) and stay valid for their own lifetime regardless.

## Security considerations

- **The assertion is short-lived by design** (`expiration` defaults to
  `2m`) — presented once, immediately exchanged, so a leaked assertion has
  a very small validity window.
- **The minted access token's lifetime is independent** (`expiration`
  defaults to `30m`) — cache and reuse it until it expires, don't
  re-exchange per call.
- **No refresh-token concept exists for `type: 'api'` sessions** —
  `SESSION_HEADERS.api.token` stays `undefined` on purpose.
- Both keys involved (the calling service's assertion key, and
  `@zanix/auth`'s own `JWK_PRI`/`JWK_PUB` behind the minted token) rotate
  fully independently, each with its own overlap window — rotating one
  never invalidates anything tied to the other.

## Checklist before adding/reviewing a service-credential integration

- [ ] Is the private key genuinely PKCS#8 and base64-encoded — not a raw
      multi-line PEM, and not a PKCS#1 key from `openssl genrsa`?
- [ ] Are permissions/rate limits set only via
      `SERVICE_PERMISSIONS_<serviceId>`/`SERVICE_RATE_LIMIT_<serviceId>` —
      never accepted from anything the caller's assertion carries?
- [ ] Does the exchange endpoint itself have no role/permission gate — trust
      comes entirely from assertion verification, not from a guard in front
      of it?
- [ ] If rotating a key, is the `_<keyId>`-suffixed form actually in use —
      the bare single-key form has no overlap window to rotate through?

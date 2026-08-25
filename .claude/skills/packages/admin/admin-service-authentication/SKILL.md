---
name: admin-service-authentication
description: The three-step service-to-service auth flow (sign an assertion, exchange it at /admin/service-token, call the real endpoint) every admin surface exposes, ZanixAdminHub.start({auth}) as the automated shortcut, calling ZanixAdminHub's own endpoints (the reverse direction), and key rotation. Use when one service needs to authenticate against another's /admin/* API or against ZanixAdminHub itself.
---

Covers the concrete, end-to-end walkthrough of `@zanix/auth`'s
service-credential exchange from this package's own routes. For the
underlying primitive (`createServiceAssertion`/`exchangeServiceCredential`,
key formats, full rotation runbook), see `@zanix/auth`'s
`auth-service-credential`. File:line references point at
`~/Documents/Development/ZanixLibraries/admin` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- Use `ZanixAdminHub.start({ auth: { serviceId } })` for the common
  outbound case — hand-roll the sign/exchange/cache plumbing only when
  building a custom client factory outside `start()`.
- Remember the two directions are configured independently — the hub
  authenticating outbound to registered services has no effect on how an
  external caller authenticates inbound to the hub, and vice versa.

## The flow, in three steps

```ts
import { createServiceAssertion } from 'jsr:@zanix/auth@[version]'

// 1. Sign, using YOUR OWN service's private key (never @zanix/auth's JWK_PRI).
const assertion = await createServiceAssertion({ serviceId: 'billing' }) // privateKey omitted -> resolves JWK_PRI_billing

// 2. Exchange it against the target service's own admin server.
const response = await fetch('http://target.internal:8000/admin/service-token', {
  method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ assertion }),
})
const { accessToken } = await response.json()

// 3. Call the real endpoint, authenticated.
await fetch('http://target.internal:8000/admin/triggers', {
  headers: { 'X-Znx-Authorization': `Bearer ${accessToken}` },
})
```

`/admin/service-token` has no role gate itself — trust is established
entirely by key verification, since the caller has no session yet (the
whole point of calling it is to get one). The receiving side needs two env
vars for the caller to be trusted at all:

```env
JWK_PUB_billing=<billing's RSA public key, base64>
SERVICE_PERMISSIONS_billing=zanix:admin:triggers
```

**Without `SERVICE_PERMISSIONS_<serviceId>`, the exchange still succeeds**
(the assertion is valid) but the minted token carries no permissions —
every role-gated route still rejects it. Permissions are never requested
by the caller; only the receiving side's own env var grants them.

## Calling a business service's own local admin API

The most common case: one service's `TriggersAggregator`/
`RemoteTemplateBackend`-style code reaching another service's own
`/admin/triggers`/`/admin/templates`. **If calling this from
`ZanixAdminHub.start()` itself, don't hand-roll any of the above** — pass
`auth: { serviceId }` to `start()` instead (below). Build the factories
manually only for something that shortcut doesn't cover: a custom
`RestClient` subclass, or different credentials for CRUD vs. Discovery
reads. Use `TriggersAdminClient`/`TemplatesAdminClient`
(`admin-triggers-aggregator`) rather than hand-rolling `fetch` — they
already speak the right protocol version/headers.

## `ZanixAdminHub.start({ auth })`: the automated shortcut

```ts
await ZanixAdminHub.start({
  rest: { port: 9000 },
  validateRegistry: true,
  auth: {
    serviceId: 'zanix-admin-hub', // privateKey/keyId omitted -> resolve JWK_PRI_/JWK_ID_zanix-admin-hub
  },
})
```

`ZanixAdminHub`'s own outbound identity for authenticating against every
service in the `ServiceRegistry` — internally signs+exchanges+caches a
credential per target via `@zanix/auth`'s `createServiceAuthClient`
(adapted here by `createServiceRegistryAuthHeaders`), installing the
resulting `TriggersAggregator`/`TemplatesDiscoveryClientFactory`
automatically. **Without `auth`, the hub's fan-out calls go out
unauthenticated** — only viable against a target that doesn't actually
require a token. Every registered service needs `JWK_PUB_zanix-admin-hub`
and `SERVICE_PERMISSIONS_zanix-admin-hub` set on **its own** process — same
trust configuration as the manual pattern, just automated.

To rotate this identity's key: register
`JWK_PRI_zanix-admin-hub_v2`/`JWK_PUB_zanix-admin-hub_v2` alongside the
current ones, then flip `JWK_ID_zanix-admin-hub` to `v2` — a config
change, not a code change, with a real overlap window
(`auth-service-credential`'s own rotation runbook).

## The reverse direction: calling `ZanixAdminHub`'s own endpoints

A caller (a deploy script, an ops tool, another service) reaching the
hub's own `/triggers`/`/templates` works exactly the same three-step way,
since `ZanixAdminHub` always registers `/admin/service-token` too. The
hub's own process needs `JWK_PUB_ops-tool`/`SERVICE_PERMISSIONS_ops-tool`
set for that caller's identity.

**This is the opposite direction from `ZanixAdminHub.start({ auth })`** —
that option only configures how the hub authenticates *outbound* to each
registered service; it has no effect on how an external caller
authenticates *inbound* to the hub, which always works this same way
regardless of whether `auth` is configured.

## Checklist before wiring service-to-service auth

- [ ] Is `ZanixAdminHub.start({auth})` used for the outbound case rather
      than hand-rolled sign/exchange/cache plumbing, unless a real
      unmet need justifies the manual path?
- [ ] Are `JWK_PUB_<serviceId>`/`SERVICE_PERMISSIONS_<serviceId>` both set
      on the receiving side — a missing permissions var lets the exchange
      succeed but leaves every role-gated route rejecting the token?
- [ ] Is the direction (hub→service vs. caller→hub) correctly identified —
      `start({auth})` only affects the outbound side, never the inbound?

---
name: auth-oauth2
description: GoogleOAuth2Connector, extending the OAuth2Connector base class for a custom provider, and why the authorization-code flow (authenticateWithCode) is the safer default over the implicit flow (authenticate) — plus the real signature difference between the provider-bound connector and a standalone instance. Use when adding/reviewing an OAuth2 login flow.
---

Covers `@zanix/auth`'s OAuth2 connectors. For JWT/session mechanics underneath
`authenticate()`/`authenticateWithCode()`, see `auth-jwt-and-sessions`; for the
zero-config provider wiring both rely on, see that skill's "Core registration"
section. File:line references point at
`~/Documents/Development/ZanixLibraries/auth` — read the real code there before
assuming this summary is still accurate.

## Golden rule (token savings)

- Default to `authenticateWithCode()` for any new OAuth2 integration — don't
  re-derive the code-vs-implicit-flow trade-off from prose each time; it's
  settled below.
- A custom provider only needs endpoints + `defaultScope` + `getSubject()` —
  everything else (`generateAuthUrl`, `getUserInfo`, `revokeToken`,
  `authenticate`, `exchangeCode`, `authenticateWithCode`, `validateCode`) comes
  free from `OAuth2Connector`, **except when a provider's real revoke contract
  genuinely differs from `revokeToken`'s own built-in assumption**
  (`POST revokeUrl`, form-encoded `token` param, no auth) — confirmed real:
  `GitHubOAuth2Connector` overrides `revokeToken` because GitHub's real endpoint
  needs `DELETE` + HTTP Basic auth (`clientId:clientSecret`) + a JSON body keyed
  `access_token`, not `token`. Verify the FULL revoke contract (method, auth,
  body shape) against the provider's own current docs during Evidence — not just
  the URL — before assuming it's free.

## `GoogleOAuth2Connector`, and the two ways to reach it

```ts
class LoginInteractor extends ZanixInteractor {
  public async auth() {
    const connector = this.providers.get('auth').google
    const { code } = /* ...from the OAuth2 redirect... */
    const { user, session } = await connector.authenticateWithCode(code, { permissions: ['admin'] })
    return session.accessToken
  }
}
```

Zero-config registration happens automatically once `jsr:@zanix/auth/core` is
imported and `GOOGLE_OAUTH2_CLIENT_ID` is set — see `auth-jwt-and-sessions`'s
Core registration section.

**Real signature difference, easy to get wrong**:
`this.providers.get('auth').google` is a `ctx`-bound wrapper —
`authenticateWithCode(code, sessionOptions?)`, no `ctx` argument. A
**standalone** `OAuth2Connector`/`GoogleOAuth2Connector` instance (constructed
directly, not reached via the provider) needs `ctx` explicit:
`connector.authenticateWithCode(ctx, code, sessionOptions?)`. Verified against
`modules/providers/extensions/google.ts` (the provider wrapper) vs.
`modules/connectors/oauth2.ts` (the base class) — don't assume one signature
works for both access paths.

## Extending `OAuth2Connector` for a custom provider

Two real, shipped precedents now exist — read them directly rather than
re-deriving the shape from prose: `GoogleOAuth2Connector`
(`connectors/google/mod.ts`, implicit-flow default, no `revokeToken` override)
and `GitHubOAuth2Connector` (`connectors/github/mod.ts`, code-flow-only default
since GitHub has no implicit flow at all, DOES override `revokeToken` — see the
Golden rule above). The general shape:

```ts
import {
  OAuth2Connector,
  type OAuth2ConnectorOptions,
} from "jsr:@zanix/auth@[version]";

type ExampleUserInfo = { id: number; email: string };

class ExampleOAuth2Connector extends OAuth2Connector<ExampleUserInfo> {
  constructor(options: OAuth2ConnectorOptions = {}) {
    super({
      authUrl: "https://example.com/oauth2/authorize",
      userInfoUrl: "https://api.example.com/user",
      revokeUrl: "https://api.example.com/oauth2/revoke",
      tokenUrl: "https://example.com/oauth2/token",
      defaultScope: "read:user",
      responseType: "code",
    }, options);
  }

  protected getSubject(user: ExampleUserInfo): string {
    return user.email;
  }
}
```

Every endpoint URL/`defaultScope` can still be overridden per-instance via the
constructor's own `options`, on top of the subclass's defaults. Building a
custom session instead of `authenticate()`'s generic one (custom permissions, a
custom payload, your own DB writes)? `validateCode(code)` is
`getUserInfo()`/`validateToken()`'s code-flow counterpart — exchanges `code` and
returns just the user info, no session built, a direct swap for
`validateToken(token)` in that kind of flow.

**Shipping a real, second provider can make an existing illustrative doc example
stale** — confirmed real: `docs/authentication-methods.md`'s own "Adding a
Custom OAuth2 Provider" section used GitHub as its hypothetical example before
`GitHubOAuth2Connector` shipped for real; adding the real connector meant that
example silently became misleading (a "here's how you'd build this" walkthrough
for something already built). When shipping a new provider, grep the package's
own docs/README for any illustrative use of that same provider's name and update
or replace it — don't leave a tutorial example describing a feature that already
exists as if it doesn't.

## Two test tiers, and what only the second one actually proves

`google.test.ts`/`github.test.ts` (`src/@tests/unit/auth/`) replace `this.http`
wholesale with a hand-rolled mock — fast, but this bypasses `RestClient`'s own
real URL-joining/header-merging/body-encoding/response-parsing logic entirely.
It cannot catch a bug in how a connector actually builds a request — which is
exactly the shape of bug `GitHubOAuth2Connector.revokeToken()`'s override exists
to prevent (a wrong method/auth/body shape sent to a real endpoint).

`src/@tests/integration/oauth2-connectors.test.ts` is the real precedent that
closes this gap: only `globalThis.fetch` is patched (the same technique
`service-auth-client.test.ts` already used elsewhere in this package, and
`notifications`' own `withFakeFetch` convention for its outbound connectors),
letting the connector's REAL code run end to end except the actual socket —
proving the request each connector builds (URL, method, headers, body) matches
that provider's real, documented contract, not just that a hand-mocked
`this.http` was called with plausible-looking arguments.

**A new provider needs BOTH tiers, not just the unit one** — add its
`authenticateWithCode()` round trip and its `revokeToken()` shape (even when
`revokeToken` isn't overridden — confirming the base class's own default request
shape still matches THIS provider's real contract is exactly the kind of check
the unit-only mock can't do) to `oauth2-connectors.test.ts` in the same change.

## Code flow vs. implicit flow — a real security trade-off, not a style choice

**Prefer `responseType: 'code'` + `tokenUrl` + `authenticateWithCode()`** over
the `responseType: 'token'` default + `authenticate()`. The code flow exchanges
`code` for a token server-side, using the connector's own `clientSecret` — the
token it hands off is provably scoped to this app by construction.
`authenticate()`'s implicit-flow input has no such guarantee: it trusts whatever
bearer token it's given, verified only by the provider, never by this app — **a
token issued for a different OAuth2 app registered with that same provider could
be replayed here.** `OAuth2Connector` logs a warning at construction whenever a
connector is left on the implicit flow, as a standing reminder — don't silence
or ignore that warning without actually switching flows.

`responseType` isn't fixed for a connector's whole lifetime — it's also a
per-call override on `generateAuthUrl()`:

```ts
connector.generateAuthUrl({ responseType: "code" });
```

`GoogleOAuth2Connector` additionally honors `GOOGLE_OAUTH2_RESPONSE_TYPE` as its
own construction-time default — useful when the connector is registered through
`@zanix/auth/core`'s env-driven setup, with no `options` object to pass a
default into directly.

## When a provider doesn't support the implicit flow at all, fail at construction — don't just document it

Some providers (GitHub, confirmed real: "The implicit grant type is not
supported") have no implicit flow whatsoever — `responseType: 'token'` isn't a
weaker option there, it's a non-option that silently fails the first time a real
user tries to log in, far from the actual misconfiguration (an env var set once,
then forgotten). **The decided criterion**: the subclass's constructor throws
immediately when the resolved `responseType` is `'token'` AND `authUrl` is left
at its default (the provider's real endpoint) — `GitHubOAuth2Connector` is the
real precedent. **Guarded only for the default `authUrl`, not unconditionally**:
a caller who explicitly points the connector at a custom `authUrl` (a proxy that
genuinely does support the implicit flow, a test double) is making a deliberate
choice the class has no basis to second-guess, so that case opts back out of the
guard. This is construction-time-config scope only —
`generateAuthUrl({ responseType:
'token' })` as an explicit per-call override is
a different, more deliberate risk shape and isn't guarded by this pattern.

Don't add this guard defensively to every new provider "just in case" — only
when Evidence (the provider's own real docs) actually confirms the implicit flow
isn't supported at all. A provider that genuinely supports both flows (like
Google) needs no such guard; documenting the trade-off (as this section already
does) is enough there.

## Checklist before adding/reviewing an OAuth2 flow

- [ ] Does the flow use `authenticateWithCode()` (code flow), not
      `authenticate()` (implicit flow) — or is there a real, reviewed reason to
      accept the implicit flow's weaker guarantee?
- [ ] Is the connector accessed via `this.providers.get('auth').google`
      (`ctx`-bound, no explicit `ctx` arg) or constructed standalone (needs
      explicit `ctx`) — matching the call signature actually used?
- [ ] Does a new custom provider implement only `getSubject()` and the endpoint
      config, rather than re-implementing any of the base class's free methods —
      and was `revokeToken`'s FULL real contract (method, auth, body shape)
      actually checked against the provider's own docs, not just assumed free
      because the URL was easy to find?
- [ ] If this provider already appeared as an illustrative example anywhere in
      this package's own docs/README, was that example updated (it's now
      describing something real, not hypothetical)?
- [ ] Does `src/@tests/integration/oauth2-connectors.test.ts` gain this
      provider's `authenticateWithCode()` + `revokeToken()` cases, patching
      `globalThis.fetch` only — not just the unit-tier `this.http` mock, which
      can't prove the real request shape matches the provider's real contract?
- [ ] If Evidence confirmed the provider has no implicit flow at all (not just a
      weaker one), does the constructor throw when `responseType` resolves to
      `'token'` against the default `authUrl` — rather than silently building a
      real-looking authorize URL that can't actually complete a login?
- [ ] Is `permissions` (the `aud` claim) set deliberately on
      `authenticate()`/`authenticateWithCode()` — see
      `auth-permissions-and-rate-limiting` for how it's later checked?

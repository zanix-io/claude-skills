---
name: auth-network-security
description: IpAllowlistGuard/@IpAllowlistGuard — IP/CIDR restriction, why trustProxyHeader must be explicitly true (client-controlled headers otherwise), the trusted-header resolution order, and combining it with AuthTokenValidation as defense-in-depth, never a replacement for it. Use when restricting a controller/route to a set of client IPs.
---

Covers `@zanix/auth`'s IP-allowlisting layer — a network-level restriction
meant to complement authentication/authorization, never replace it. File:line
references point at `~/Documents/Development/ZanixLibraries/auth` — read the
real code there before assuming this summary is still accurate.

## Golden rule (token savings)

- Always pair `@IpAllowlistGuard` with `@AuthTokenValidation` on a sensitive
  controller — an allowlist alone is not authentication.
- Only enable `trustProxyHeader: true` when the deployment genuinely runs
  behind a trusted reverse proxy that overwrites client-IP headers — verify
  that once per deployment, don't assume it.

## Basic usage

```ts
import { AuthTokenValidation, IpAllowlistGuard } from 'jsr:@zanix/auth'

@Controller()
@AuthTokenValidation({ permissions: ['admin'] })
@IpAllowlistGuard({
  allow: ['10.0.0.0/8', '192.168.1.25'],
  trustProxyHeader: true,
})
export class AdminController {}
```

A request from outside the configured allowlist gets a `403 Forbidden`.
Both exact IPs and IPv4 CIDR ranges are supported (`10.0.0.0/8` matches any
`10.x.x.x`).

## `ADMIN_IP_ALLOWLIST` env-var fallback

```env
ADMIN_IP_ALLOWLIST=10.0.0.0/8,192.168.1.25,203.0.113.15
```

When `allow` is omitted, the guard falls back to this env var. **If neither
`allow` nor `ADMIN_IP_ALLOWLIST` is configured, the guard becomes a
pass-through and doesn't restrict requests at all** — an empty/unset
allowlist is not the same as "deny everything," it's "allow everything."

## `trustProxyHeader` — a mandatory, explicit acknowledgment

HTTP proxy headers (`x-forwarded-for`, `cf-connecting-ip`, `x-real-ip` — all
three trusted by default) are client-controlled unless the deployment's own
infrastructure overwrites them. **For this reason, an allowlist cannot be
enabled unless `trustProxyHeader` is explicitly `true`** — attempting to
configure `allow`/`ADMIN_IP_ALLOWLIST` without it throws `InternalError` at
application startup, not silently ignoring the allowlist. Only set it `true`
when requests genuinely pass through a trusted reverse proxy (Cloudflare,
NGINX, Traefik, HAProxy, Kubernetes Ingress, AWS ALB/ELB) configured to
overwrite client-IP headers before forwarding — setting it `true` behind an
untrusted or absent proxy makes the allowlist trivially bypassable by anyone
who forges the header themselves.

Restrict which headers are trusted when the infrastructure guarantees a
specific one:

```ts
@IpAllowlistGuard({
  allow: ['203.0.113.5'],
  trustProxyHeader: true,
  trustedHeaders: ['cf-connecting-ip'],
})
```

## `IpAllowlistOptions`

```ts
interface IpAllowlistOptions {
  allow?: string[]
  trustProxyHeader?: boolean
  trustedHeaders?: string[]
}
```

Available both as a class decorator (`@IpAllowlistGuard(options)`) and a
plain middleware guard (`ipAllowlistGuard(options)`) for manual
registration.

## Checklist before adding/reviewing an IP allowlist

- [ ] Is `trustProxyHeader: true` set only because this deployment's real
      infrastructure overwrites client-IP headers — not copied from another
      project without checking?
- [ ] Is the allowlist paired with `@AuthTokenValidation` (or another real
      authentication check) on the same controller, not standing alone as
      the only access control?
- [ ] If relying on `ADMIN_IP_ALLOWLIST`, is it actually set — an unset
      allowlist (env var and `allow` both missing) silently disables
      restriction entirely?
- [ ] If `trustedHeaders` is narrowed, does it match what the actual proxy
      in front of this deployment sets?

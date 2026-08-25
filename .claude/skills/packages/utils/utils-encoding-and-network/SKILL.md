---
name: utils-encoding-and-network
description: Encoding helpers (base64/base64url/base32/hex, case conversion, stripComments), zanixConstants/zanixRegex, and IP/CIDR utilities (getClientIp, isIpInCidr) — including the real spoofing risk in trusting proxy headers by default, and stripComments' trusted-input-only warning. Use when encoding/decoding a value, matching against a shared regex, or resolving/validating a client IP.
---

Covers `@zanix/utils/helpers`'s encoding functions, `/constants`,
`/regex`, and the small IP/CIDR module. File:line references point at
`~/Documents/Development/ZanixLibraries/utils` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- `getClientIp`'s trusted-header order is a real spoofing surface if the
  deployment isn't actually behind the proxy that sets those headers —
  check this before trusting the returned IP for anything access-control-related.
- `stripComments` is for trusted JSONC-style config only — never reach for
  it to sanitize untrusted code/text.

## Encoding

```ts
const encoded = base64UrlEncode('Hello, World!') // 'SGVsbG8sIFdvcmxkIQ' — no '+', '/', or '=' padding
const decoded = base64UrlDecode(encoded, true) // 'Hello, World!'

hexToUint8Array('48 65') // Uint8Array [0x48, 0x65] — spaces tolerated
hexToUint8Array('abc') // throws: Hex string must have an even length
```

`base64UrlEncode`/`base64UrlDecode` — URL-safe base64 (`+`→`-`, `/`→`_`,
padding stripped), the shape JWTs/URL tokens need. `base32Encode`/
`base32Decode` — RFC 4648, uppercase `A-Z2-7`, no padding (TOTP/
authenticator-app convention); decode tolerates lowercase and optional `=`
padding. `hexToUint8Array` throws a specific error on odd-length hex — a
real signal for malformed/truncated input, not a silent partial parse.
`compareUint8Arrays(a, b)` — byte-by-byte, returns `false` immediately on a
length mismatch.

**Case conversion, two distinct pairs, not interchangeable**:
`capitalize`/`capitalizeWords` (prose capitalization — first char only, or
first char of each space-separated word) vs. `toKebabCase`/`toPascalCase`
(identifier-casing conversion, handling camelCase/PascalCase/snake_case/
spaced input, splitting consecutive capitals as their own word —
`'XMLParser'` → `'xml-parser'`). `toPascalCase('PAYMENT')` → `'Payment'`
(lowercases first, then capitalizes) — `capitalizeWords('PAYMENT')` would
leave it `'PAYMENT'` unchanged. Don't reach for one pair expecting the
other's behavior.

**Real footgun**: `stripComments(value)` strips `//`/`/* */` from
JSONC-like strings without touching comment-like text inside quotes — the
doc's own explicit warning: "use it with trusted input only, it is meant
for JSONC-style config files, not for sanitizing untrusted code."

`isZanixHex(str)` matches `/^Zx[0-9a-fA-F]+$/`. `stringToUint8Array`/
`uint8ArrayToString`/`uint8ArrayToBase64`/`base64ToUint8Array`/
`uint8ArrayToHEX` round out the byte-array conversion surface; `encoder`/
`decoder` export shared `TextEncoder`/`TextDecoder` instances.

## Constants and regex

`zanixConstants` (frozen default export, `/constants` subpath) — includes
`CONFIG_FILE` (`'deno.json'`), `DISTRIBUTION_FILE`, `MAIN_MODULE`,
`ZNX_FLAGS` (`['use comet']` — what `utils-linter-plugins`'s
`use-znx-flags` rule validates against), `ZANIX_LOGO`.

`zanixRegex` (frozen default export, `/regex` subpath) — `EMAIL_REGEX`,
`URL_REGEX`, `UUID_REGEX`, `PHONE_REGEX`, `SECURE_PASSWORD_REGEX`,
`USERNAME_REGEX`, `ISO_DATE_REGEX`, `VERSION_REGEX`, and others; these are
the same patterns `utils-validator-decorators`'s `IsEmail`/`IsUrl`/`IsUUID`/
`IsPhone` decorators use internally.

**Real gotchas worth knowing before relying on one**:
- `PHONE_REGEX` rejects spaces entirely — `'+1 234 567 890'` fails; a
  formatted phone number needs stripping first.
- `SECURE_PASSWORD_REGEX` requires lowercase + uppercase + digit (min 8
  chars); special characters are allowed but not required.
- `ISO_DATE_REGEX` checks format only, not calendar validity — `2025-02-30`
  still matches.
- `UUID_REGEX` validates UUID **v4** specifically, not any UUID version.

## IP/CIDR utilities

```ts
import { getClientIp, isIpInCidr, normalizeClientIp } from 'jsr:@zanix/utils@[version]/helpers'

const ip = getClientIp(request.headers)
isIpInCidr(ip, '10.0.0.0/8') // true | false
normalizeClientIp('::ffff:192.168.1.10:8080') // '192.168.1.10'
```

`getClientIp(headers, trustedHeaders?)` extracts a client IP from the
first trusted header present, checked in order:
`cf-connecting-ip`/`x-forwarded-for`/`x-real-ip` (all three trusted by
default) — returns the string `'unknown-ip'`, not `undefined`, when none
are present.

**Real security-relevant footgun**: these headers are client-controlled
unless the deployment's own infrastructure genuinely overwrites them
before the request reaches this code. Trusting `getClientIp`'s default
header order without restricting `trustedHeaders` to match the actual
proxy chain in front of the app is spoofable — the same class of risk
`auth-network-security`'s `IpAllowlistGuard` addresses at the guard layer
(that guard requires `trustProxyHeader: true` explicitly for exactly this
reason; `getClientIp` itself has no equivalent opt-in gate, so the caller
is responsible for restricting `trustedHeaders`).

`ipv4ToInt`/`parseCidr` return `undefined` on invalid input rather than
throwing — a silent-failure shape worth checking for explicitly rather
than assuming a non-`undefined` result. IPv4-only — no IPv6 CIDR/range
support (only IPv4-mapped IPv6 normalization for the address itself, via
`normalizeClientIp`, which also trims whitespace and strips a port
number).

## Checklist before using an encoding/regex/IP utility

- [ ] Is `stripComments` applied only to trusted config content, never to
      anything reachable by untrusted input?
- [ ] Is the right case-conversion pair used — `capitalize*` for prose,
      `to*Case` for identifiers — not assumed interchangeable?
- [ ] Is `getClientIp`'s `trustedHeaders` actually restricted to match the
      real proxy chain in front of this deployment — not left at the
      permissive default when the app isn't genuinely behind a trusted
      proxy?
- [ ] Does code checking `ipv4ToInt`/`parseCidr`'s result explicitly guard
      for `undefined`, rather than assuming a valid parse?

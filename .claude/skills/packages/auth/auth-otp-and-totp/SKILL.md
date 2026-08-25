---
name: auth-otp-and-totp
description: OTP (one-time password, generated/delivered/verified server-side) and TOTP (authenticator-app 2FA, HMAC-SHA1/6-digit/30s-step) — both a generate-then-verify shape, each with a combined .authenticate() that also creates the local session. Use when adding a second-factor or delivery-based one-time-code flow.
---

Covers `@zanix/auth`'s two one-time-code mechanisms — OTP (you deliver the
code yourself, e.g. email/SMS) and TOTP (an authenticator app generates the
code). For the session `authenticate()` produces, see
`auth-jwt-and-sessions`. File:line references point at
`~/Documents/Development/ZanixLibraries/auth` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- Both mechanisms share one shape: generate → deliver/persist yourself →
  verify. Don't re-derive that shape per mechanism; the only real
  differences are what's generated and where it's stored.
- Trust the TOTP defaults (HMAC-SHA1, 6 digits, 30s steps) — they're what
  every real authenticator app expects. Only the HMAC algorithm is actually
  hardcoded in `totp.ts`; `digits`/`period` are real accepted parameters on
  `generateTOTP()`/`verifyTOTP()`/`getTOTPProvisioningUri()` (each carries a
  JSDoc warning against changing them, not a hard block) — don't claim
  they're non-configurable, just strongly discouraged to change.

## OTP (One-Time Password)

```ts
class LoginInteractor extends ZanixInteractor {
  public async requestOtp(user: { email: string }) {
    const code = await this.providers.get('auth').otp.generate({ target: user.email })
    // Send `code` to the user yourself (email/SMS) — the library only generates and caches it.
    return { sent: true }
  }

  public async verifyOtp(user: { email: string }, code: string) {
    const session = await this.providers.get('auth').otp.authenticate(
      user.email,
      code,
      { permissions: ['admin'] }, // becomes the session token's `aud` claim
    )
    return session.accessToken
  }
}
```

The code is cached server-side (Redis when `REDIS_URI` is set, otherwise
in-memory) against `target`, expires after `exp` seconds (default `300`),
and is removed as soon as it's verified once — a code is never reusable
after a successful verification, whether or not the caller checks for that
itself.

Wrong guesses are capped too: `.otp.verify()`/`.otp.authenticate()` track
failed attempts per `target` and burn the code outright (delete it, return
`false`) once `maxAttempts` is reached (default `5`, via
`DEFAULT_OTP_MAX_ATTEMPTS`) — even if the code hasn't actually expired yet.
This is what stops a short numeric code from being brute-forced across its
own TTL.

`.otp.generate()`/`.otp.verify()` are the low-level pair;
`.otp.authenticate()` combines verification with session creation (mirrors
`.google.authenticate()`/`.totp.authenticate()`).

## TOTP (Authenticator App 2FA)

```ts
class TwoFactorInteractor extends ZanixInteractor {
  public enroll(user: { email: string }) {
    const totp = this.providers.get('auth').totp
    const secret = totp.generateSecret()
    const uri = totp.getProvisioningUri(secret, user.email, { issuer: 'MyApp' })
    // Render `uri` as a QR code for the user to scan, and persist `secret` yourself
    // (e.g. on the user record) — the library never stores it for you.
    return { uri }
  }

  public async verify(user: { email: string; totpSecret: string }, code: string) {
    const session = await this.providers.get('auth').totp.authenticate(
      user.totpSecret,
      code,
      { subject: user.email, permissions: ['admin'] },
    )
    return session.accessToken
  }
}
```

**The secret is never persisted by this package** — `generateSecret()`
returns it and the caller is responsible for storing it (typically on the
user record) and passing it back into `verify()`/`authenticate()`.
`issuer` defaults to `'zanix-auth'` when omitted.

Defaults for `generateTOTP()`/`verifyTOTP()`/`getTOTPProvisioningUri()`:
HMAC-SHA1, 6-digit codes, 30-second steps — the values every real
authenticator app (Google Authenticator, Microsoft Authenticator, etc.)
expects. **Only the HMAC algorithm is actually hardcoded** in `totp.ts` —
`digits`/`period` are real accepted parameters on all three functions, each
with a JSDoc warning against changing them (real authenticator apps assume
6/30s, changing either breaks compatibility with them), not a hard block.
Don't tell a caller they're non-configurable; tell them why changing them
is a bad idea. `verifyTOTP()` accepts the previous/current/next step by
default, to tolerate real clock drift between the server and the user's
device.

## Checklist before adding an OTP/TOTP flow

- [ ] Is the code actually delivered (email/SMS) by the caller — this
      package only generates and caches it, it never sends anything?
- [ ] For TOTP, is the secret persisted by the caller (on the user record),
      never assumed to be stored by `@zanix/auth`?
- [ ] Is `permissions` set deliberately on `.authenticate()` — see
      `auth-permissions-and-rate-limiting` for how it's checked later?
- [ ] Is the combined `.authenticate()` used when a session should be
      created immediately on successful verification, vs. the standalone
      `.verify()` when verification and session creation genuinely need to
      be separate steps?

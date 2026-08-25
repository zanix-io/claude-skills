---
name: utils-errors
description: HttpError/ApplicationError/PermissionDenied/InternalError's shared options and asymmetric shouldLog defaults, serializeError/serializeMultipleErrors, and the redact-by-default behavior shared with the Logger. Use when throwing/catching an ecosystem error, or serializing one for a response/log.
---

Covers `@zanix/utils/errors` — the error hierarchy every Zanix package
throws through. For how these get auto-logged and the shared redaction
mechanism, see `utils-logger`. File:line references point at
`~/Documents/Development/ZanixLibraries/utils` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- Check the `shouldLog` default table before assuming uniform behavior
  across the hierarchy — `InternalError` is the one asymmetric default.
- Trust `serializeError`'s redaction-by-default — reach for `redact: false`
  only when the output is already fully trusted (never receives
  request/header data or user input).

## The hierarchy

`HttpError extends Deno.errors.Http` (falls back to plain `Error` outside
Deno — e.g. inside a `@zanix/space` Comet running in a browser — with
identical `.message`/`.status`/`.stack`/`.cause`/`.meta`/`.code` behavior
either way, since all are set in the error's own constructor).
`ApplicationError extends Error`. `PermissionDenied extends
ApplicationError`. `InternalError extends ApplicationError`.

Shared options (`ErrorOptions`): `message`, `shouldLog` (opt-in logging via
`@zanix/utils/logger` on construction), `meta`, `code`, `cause`, `id`
(default generated UUID).

| Class | `shouldLog` default |
| --- | --- |
| `HttpError` | `false` |
| `ApplicationError` | `false` |
| `PermissionDenied` | `false` |
| `InternalError` | **`true`** — the one asymmetric default in the hierarchy |

```ts
throw new HttpError('NOT_FOUND', {
  message: 'User not found',
  shouldLog: true,
  meta: { userId: '12345' },
})

const error = new HttpError('BAD_REQUEST', { message: 'Invalid input provided.' })
error.status.code  // 'BAD_REQUEST'
error.status.value  // 400

throw new InternalError('Invalid input provided.', { shouldLog: false }) // opt out of the default
```

`error.status` (`HttpError` only) is `{code, value}`, backed by
`httpStates` — a plain map from every `HttpErrorCodes` name to its numeric
status, covering the common 4xx/5xx set.

## `serializeError`/`serializeMultipleErrors`

```ts
const withCustomPattern = serializeError(error, { redact: { pattern: /^internal-id$/i } })
const withoutRedaction = serializeError(error, { redact: false }) // only if output is already trusted
```

`serializeError(error, options?)` converts an error or unknown value into a
plain serializable object. For a real `Error` instance: extracts
`name`/`message`/`stack` (unless `withStackTrace: false`) and recursively
serializes `cause`. For anything else: `JSON.parse(JSON.stringify(...))`
as a fallback; if that itself fails, returns the original input as-is
rather than throwing.

**Redaction is on by default and recursive** through `cause` chains and
`meta` — credential-shaped fields (matching the same pattern
`utils-logger` documents) are redacted automatically. `redact: false`
bypasses this entirely — only safe when the output is already fully
trusted, never when it might carry request/header data or user input.

`serializeMultipleErrors(errors, options?)` applies `serializeError` over
an array, forwarding `withStackTrace`/`redact` to every entry — this is
what `logger.error(message, ...data)` uses internally. It skips an error
already serialized once (tracked via an internal non-enumerable `_logged`
marker), so the same error instance flowing through multiple `logger.error`
calls doesn't get logged twice.

## Checklist before throwing or serializing an error

- [ ] Is `shouldLog` set deliberately — not assumed to default the same
      way across `HttpError`/`ApplicationError`/`PermissionDenied` vs.
      `InternalError`?
- [ ] Is `redact: false` used only when the serialized output is genuinely
      never exposed to request/header/user-input data?
- [ ] Does code running outside Deno (a `@zanix/space` Comet) still expect
      `HttpError`'s `.status`/`.meta`/`.code` to behave identically, even
      though its base class differs there?

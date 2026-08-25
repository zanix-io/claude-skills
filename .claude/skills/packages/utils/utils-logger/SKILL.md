---
name: utils-logger
description: The default Logger instance and its six log levels (including logger.high, between warn and error), the six storage configuration styles, redact/setDefaultRedactOptions (on by default, recursive), and real footguns — logger.error's silent dedup-to-nothing (which logger.high does not share), the JSON.stringify(Error)-collapses-to-{} trap, and global-instance clobbering. Use when configuring logging, choosing a storage backend, or reasoning about what gets redacted.
---

Covers `@zanix/utils/logger`. For the error-serialization mechanics it
shares redaction defaults with, see `utils-errors`. File:line references
point at `~/Documents/Development/ZanixLibraries/utils` — read the real
code there before assuming this summary is still accurate.

## Golden rule (token savings)

- Use the default `logger` instance for the common case — reach for `new
  Logger(options)` only when this specific process genuinely needs
  different storage/redaction than the global default.
- Check the redact-pattern-vs-extend distinction before customizing —
  `pattern` replaces the built-in set entirely, `extend` adds to it.

## The default instance and basic methods

```ts
import logger, { Logger } from 'jsr:@zanix/utils@[version]/logger'

logger.info('Server started', { port: 3000 })
logger.warn('Cache miss', { key: 'user:42' })
logger.error('Failed to fetch user', someError)
logger.high('Unexpected retry exhaustion', { attempts: 5 })
logger.debug('Incoming payload', { body: requestBody })
logger.success('Migration completed')
```

Each method prints via the matching `console.*` call, prefixed with an
icon and a `ZNX-*` tag — `high` is the one exception: `console` has no
`console.high`, so it reuses `console.error`'s parameter shape and
underlying console call, with its own icon/tag (`🟣 ZNX-HIGH`) to stay
visually distinct from `error`. **`info`/`warn`/`error`/`high` are
persisted** per the configured storage strategy (default: JSON files
under `.logs`). **`debug`/`success` are never persisted** — console
only, regardless of storage config.

## `logger.high`: the sixth level, between `warn` and `error`

`logger.high(...data)` logs an anomalous condition that deserves
attention sooner than a routine `warn`, but where the operation itself
didn't necessarily fail outright (unlike `error`) — e.g. retry
exhaustion that still eventually succeeded, or a degraded-but-serving
state. It takes the same variadic shape as `warn`/`error`.

**Real footgun — no dedup**: unlike `logger.error`, `high` does **not**
perform the `_logged` dedup described below. Passing the same `Error`
object to `high` on repeated calls prints/persists every time — pass an
`Error` to `high` as plain extra data, not expecting `error`'s
once-only behavior.

## Storage: six configuration styles

```ts
const logger = new Logger({ storage: { save: { folder: 'myCustomFolder', expirationTime: '1d' } } })
```

1. **Custom function**: `save(context)` (sync or async) —
   `context.getFmtLog()` returns the formatted log (includes an ISO-8601
   `timestamp` by default).
2. **File-based**: `save: { folder?: string (default '.logs'),
   expirationTime?: string (default '5d') }`.
3. **Worker offload**: `save: { useWorker: true, callback?: () => void }`
   — runs the save inside a one-time `WorkerManager` worker
   (`utils-workers`).
4. **`formatter: (level, logData) => custom shape`** — reshapes a log
   before it reaches save/file, compatible with any storage style.
5. **Disable persistence**: `storage: false` (whole instance) or pass
   `'noSave'` as the last argument on a single call.
6. **Reusable backend factory**: `(options) => SaveDataFunction` — not a
   special shape `Logger` recognizes, just a convention; real example:
   `@zanix/datamaster`'s `elasticsearchLogSave` (its `/observability`
   subpath).

**Real footgun, explicitly documented**: `Error` instances have
non-enumerable own properties, so `JSON.stringify(someError)` silently
collapses to `{}` — a real trap for any custom storage strategy that
JSON-encodes the formatted log as-is. `Logger` works around this
internally by serializing errors through the same logic as `serializeError`
(`utils-errors`) before formatting — a custom `save` function receiving
raw error objects needs the same care.

**Real footgun**: `Headers`/`Request` objects serialize to `{}` under
`JSON.stringify` but Deno's console inspector prints their full contents
(`Authorization` included) — `Logger` special-cases these into safe named
fields (`method`/`url`/`headers`) before redaction, precisely to avoid a
credential leak through the console-vs-JSON discrepancy.

## Redaction: on by default, recursive

```ts
const logger = new Logger({ redact: { extend: ['dbPassword', /secret$/i] } })
const customLogger = new Logger({ redact: { pattern: /^(authorization|x-internal-.*)$/i } })

logger.warn('Login attempt', { token: 'abc123', headers: someRequest.headers })
// Login attempt { token: '[REDACTED]', headers: { authorization: '[REDACTED]', ... } }
```

Default pattern matches (case-insensitive): `authorization`, `cookie`,
`password`, `token`, `secret`, `apiKey`, and similar. `redact: false`
disables it — the doc's own warning: "only do this if this logger's output
is already fully trusted (e.g. it never receives request/header data or
user input)."

**Real footgun — `pattern` replaces, `extend` adds**: `redact.pattern`
**replaces the built-in set entirely** rather than extending it, in
contrast to `redact.extend` which adds to it — easy to accidentally lose
the built-in credential-key protection by reaching for `pattern` when
`extend` was the intent.

**Two separate redaction knobs, don't conflate them**: a per-`Logger`
`redact` option does **not** affect `serializeError`'s own fallback
behavior used elsewhere in the codebase (e.g. by `@zanix/server` building
client-facing error responses) — that shared default is controlled
separately via `setDefaultRedactOptions(options)` (from
`jsr:@zanix/utils/errors`; `setDefaultRedactOptions(false)` disables the
fallback). An explicit `redact` (on a `Logger` instance, or passed
directly to `serializeError`) always wins over that shared default either
way.

## `logger.error`'s silent dedup

`logger.error(...)` marks each error internally (`_logged`) the first time
it's serialized. Passing the exact same error object again filters that
duplicate; **if every extra argument in a call is already a marked
duplicate, the entire call is silently skipped** — nothing printed or
saved. Worth knowing before assuming a repeated `logger.error(sameError)`
call is a no-op bug rather than deliberate dedup.

## Global instance assignment

Creating `new Logger()` stores the instance on `globalThis`/the
`Zanix`/`Znx` namespace (`Znx.logger`, `self.logger`) unless
`disableGlobalAssign: true` is passed — **the most recently created
instance silently overwrites the previous global reference**. An app
creating multiple `Logger` instances for different purposes should pass
`disableGlobalAssign: true` on all but the one meant to be the process-wide
default, or every earlier instance's global accessibility gets clobbered.

**The module itself already does this once, at import time.**
`disableGlobalAssign` only ever applies to a `Logger` instance YOU
construct — it can't retroactively undo the module's own internal
`new Logger()`, which runs unconditionally the moment `@zanix/utils/logger`
is imported (`modules/logger/mod.ts`), and whose constructor seeds `Znx`
via `setGlobalZnx` regardless of any option (`modules/logger/main.ts`) —
see `utils-config-and-project-helpers`'s own note on the `Zanix`/`Znx`
namespace for the full mechanics. Importing the module alone is enough to
seed the global; it doesn't require your own `new Logger()` call first.

## Checklist before configuring or extending logging

- [ ] Is a custom `save` function's log data actually run through
      `serializeError`-equivalent handling before `JSON.stringify`, or
      will a raw `Error` silently collapse to `{}`?
- [ ] Is `redact.extend` used to add patterns, reserving `redact.pattern`
      only for a deliberate full replacement of the built-in set?
- [ ] If this deployment needs a different default `serializeError`
      redaction than the built-in one, is `setDefaultRedactOptions` used —
      not a per-`Logger` `redact` option, which doesn't reach that shared
      path?
- [ ] Does a new `Logger` instance need `disableGlobalAssign: true` to
      avoid silently replacing the process-wide `Znx.logger` reference?

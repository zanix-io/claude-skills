---
name: zanix-observability-conventions
description: How to log/throw/persist correctly across the Zanix ecosystem — which log level and 'noSave', which of the 4 shared error classes, message vs userMessage, what actually reaches an HTTP client, the connector-error sanitization discipline, and whether a new sensitive `X-Znx-` header/cookie needs registering in `SENSITIVE_KEY_PATTERN`/`redact.extend`. Grounded in a full 11-repo observability audit. Use when writing/reviewing code that logs an event or throws an error, or when a change introduces a new credential-carrying header/cookie, in a library repo or a consumer project. Does NOT cover Logger/error-hierarchy internals — see utils-logger/utils-errors for that.
---

This is about **using** `@zanix/utils`'s logging/error primitives correctly,
not how they're built — see `utils-logger`/`utils-errors` for the
implementation. Grounded in a full 11-repo audit ("Zanix Observability
Audit", 2026-08-20) that found the shared architecture itself was already
sound; the real inconsistency was in the DECISIONS every subrepo made
independently on top of it. File:line references point at
`~/Documents/Development/ZanixLibraries/<repo>` — read the real code there
before assuming this summary is still accurate; several of the fixes cited
here are recent and some are still blocked on a JSR publish (noted where
that applies).

## Golden rule (token savings)

- **Two separate questions, don't conflate them**: "does this print to
  console" (yes, always, for every level) vs. "does this persist" (`'noSave'`
  — see the rule below, this is where the real inconsistency was). Auditing
  "is this logged" without asking which of the two you mean produces a false
  sense of consistency.
- **Classifying an error is one question**: did the CALLER cause this, or
  something outside their control? Check the class table below before
  reaching for a generic `Error` or a standalone class.
- This skill doesn't own general test-coverage completeness — that's
  `feature-completeness-conventions`'s gate, already mandatory. What this
  skill adds on top: tests for logging/error behavior must assert
  `instanceof <SpecificClass>` + `.code`, never a generic `Error` + message
  substring — confirmed real: most of this audit's own first-pass fixes used
  the substring form and had to be reinforced afterward because it doesn't
  actually guard against regression.

## The `'noSave'` rule — the actual source of inconsistency this audit found

`debug`/`success` never persist (hardcoded). `info`/`warn`/`error`/`high` do,
by default — except when `Znx.config.project` is `'library'` or `'app'`, where
the default is a no-op (nothing persists) unless the consumer configures its
own `save`. The real, single rule to apply everywhere:

> A log persists (no `'noSave'`) if and only if it describes something an
> operator would need to be able to find without having been watching the
> console at the exact moment it happened. Mark `'noSave'` ONLY for (a)
> high-volume routine lifecycle (connect/disconnect, heartbeat, **a single
> non-terminal retry attempt**), or (b) something that's already going to be
> summarized/persisted elsewhere in the same flow. Everything else — the
> final outcome of an operation (success or failure), any `high`/`error`, any
> `warn` signaling real degradation — persists by default.

Two real, confirmed violations this rule reclassified (not just "bad taste"
— concrete rule violations): `asyncmq` marking every retry attempt,
including non-terminal ones, without `'noSave'`; `auth` marking everything
as `'noSave'`, including real outcomes that aren't part of any retry (a
silent decrypt fallback, a silent fall-through to local KV when Redis isn't
configured). Real precedent for the correct shape: `asyncmq/src/modules/
subscribers/base.ts`'s `onerror` — non-terminal attempt → `warn`+`'noSave'`,
terminal failure → `error` with no `'noSave'`.

`shouldNotLogError`/throttle (`server/src/utils/errors/helper.ts`): repeated
4xx errors below 50/hour (default) **don't log at all**, console or storage —
don't assume "there's a `logger.error` in this path" means something always
persists.

## `high` — real, in use, but check what your package can actually resolve

A real level between `warn` and `error` (`logger.high(...)`) — persists by
default same as `warn`/`error`, respecting `'noSave'` per the rule above.
See `utils-logger`'s own `logger.high` section for the full mechanics
(color/icon, the `console.error` routing, and the no-dedup footgun) — this
skill only covers when to reach for it.

Use it for "anomalous and significant, but the operation didn't outright
fail" — warn's own category, elevated because an operator shouldn't have to
go looking for it (e.g. a retry budget exhausted before falling back to a
manual/degraded mode; a security-relevant misconfiguration that still lets
the process run — `asyncmq/src/modules/rabbitmq/provider/mod.ts`'s
constructor, a real confirmed call site: warns at `high` when
`DATA_AMQP_SECRET` is unset and the process falls back to a hardcoded,
publicly-known key). Never a replacement for `error`.

**Check before assuming availability**: `high` was added to `@zanix/utils`
very recently — confirm via that package's real `CHANGELOG.md`/`git status`
whether it's landed in a published JSR version yet, or still local-only, the
same "local-ahead-of-JSR" state this ecosystem has repeatedly hit elsewhere
(see `dependency-drift`). A package resolving `@zanix/utils` from an
already-published pin that predates `high` will throw a real
`TypeError` (missing method) if you call it — not silently no-op. This
isn't a reason to avoid `high` in code moving in lockstep with utils's own
local changes (using it correctly now, ahead of publish, is the right call,
confirmed working against local source) — it's a reason to actually check
which `@zanix/utils` version the target package resolves before assuming any
package can call it today.

## Choosing the right error class

The single question: **did the problem come from whoever's calling, or from
something outside their control?**

| Class | `shouldLog` default | Use for |
| --- | --- | --- |
| `HttpError` | `false` | Outside the specific HTTP CLIENT's control, with an explicit status. |
| `InternalError` | **`true`** — the one asymmetric default | Outside the caller's control, no HTTP status — **self-logs on construction, don't call `logger.error` manually on top, that double-logs.** |
| `ApplicationError` | `false` | Caller misused the API (not necessarily HTTP). |
| `PermissionDenied` | `false` | Authorization specifically. |

Never a native `Error`, `Deno.errors.*`, or a standalone local class without
a real domain justification. Confirmed legitimate exceptions:
`SmtpConnectionClosedError` (notifications) and `MessageCompileError` (cli)
— both extend the shared hierarchy (`ApplicationError`), they don't replace
it; a class that just wraps `Error` directly loses `id`/`code`/`cause` for
no reason.

**The double-log anti-pattern, confirmed real, fixed, worth citing as the
literal precedent**: `datamaster/src/modules/database/providers/mongo/
connector/mod.ts`'s `initialize()` catch block — `InternalError` with
`shouldLog:true` self-logs at construction (stamps `_logged:true`); the old
code ALSO called `logger.error` right after constructing it, double-logging
the same failure. Fixed by removing the manual call — the comment left in
place documents the "throw-and-log-once" pattern explicitly. **The rule to
apply generally: never call `logger.error` immediately after constructing an
error with `shouldLog:true`.**

**`retryable` doesn't exist on the shared error classes yet — this audit
recommends adding it, don't cite it as already available.** It's the single
most consistent gap this audit found across all 11 repos; if you're touching
the shared error-options shape, this is a real, confirmed-needed addition,
not something to invent a local equivalent for per-package.

**Shape, settled: `retryable?: boolean`, not an enum.** Surveyed every real
retry mechanism across the ecosystem before deciding — none consults an
error's class/identity to choose HOW to retry; the one place identity
forks the decision at all (`datamaster`'s Redis connector, a specific error
`code` forcing "never retry"; `notifications`'s SMTP client, only
`SmtpConnectionClosedError` retries) is a binary fork, not multi-state.
`asyncmq`'s subscriber redelivery produces 3 real outcomes (immediate/
delayed/dead-letter) but the selection is driven entirely by attempt-count/
config, never by the error's own nature — the "how" already lives in
per-mechanism `retryConfig`, not on the error. A `boolean` models the real
pattern found; absent/`undefined` means unknown/caller-decides, matching
today's implicit default. If a delay hint is ever genuinely needed, that's
a separate `retryAfterMs`-shaped field later, not folding it into an enum
now with no real consumer to interpret it.

## `code`: naming convention, deliberately no central registry

`DOMAIN_SUBSYSTEM_REASON` — e.g. `MONGODB_CONNECTION_FAILED`,
`DATAMASTER_BULK_WRITE_RETRY_EXHAUSTED`, `REDIS_CONNECTION_TIMEOUT` (already
conformant). `admin`'s `UNKNOWN_SERVICE` (missing domain prefix) and
`notifications`'s historical lack of any own codes (generic HTTP codes
instead) are the confirmed non-conformant examples to avoid repeating.

**Deliberately not centralized** — a shared catalog would create exactly the
cross-package publish coupling `zanix-dependency-direction` already avoids
elsewhere (every new `code` would mean touching `@zanix/utils` and waiting
on its own publish cycle), for a registry that today has no real consumer
needing to iterate over "every possible code" programmatically. `code` is
required for any error crossing a public boundary — the convention is
mandatory, the registry is not.

**A boot-time/constructor error that can never reach a request/response
cycle is exempt from this — don't flag it as missing `code`.** Confirmed
against real precedent (`server`'s `WebServerManager` SSL-config fix and
~30 other `InternalError` sites across the repo, none of which set `code`):
an error thrown before the server has ever served a request — a
misconfiguration that prevents startup entirely — is dev/ops-facing only, by
construction never reaches `getPublicErrorResponse`. `meta`-only, no `code`,
is the real, consistent norm for this shape, not an oversight to correct.
Same reasoning applies to `userMessage` in the section below — the audience
table is framed around request-time errors; a startup failure has no
request to attach a user-facing message to at all.

**This exemption is narrow — it covers `code`/`userMessage` only, never the
error class itself.** A boot/build-time-only throw (never reaching a
request/response cycle) isn't held to the same hard "never a native `Error`"
rule the section above states for request-reachable code, since nothing
structurally forces it there — but using one of the four shared classes
stays the strongly preferred default even here, not merely an option, and
is what the real precedent above (`WebServerManager` and the ~30 other
sites) actually does: every one of them wraps a raw native/OS error in
`InternalError` (`meta`/`cause`, no `code`/`userMessage`) rather than
letting it propagate unwrapped. A confirmed, real misapplication of this
exemption: treating "boot-time errors don't need `code`" as "boot-time
errors don't need wrapping at all," and leaving a raw `throw error` (a
native `Deno.errors.*`) in place — that's the gap to flag, not a correctly
identified exemption.

## Messages: `message` vs `userMessage`

`message` is always dev-facing — logs, `cause` chains, responses another
developer consumes — **never assumed safe to show a real end user.**
`userMessage` is optional, with **no automatic fallback to `message`**
(better absent than accidentally leaking internal detail) — the caller sets
it explicitly only on errors that can realistically reach a UI.

**Audience differs by subrepo — don't apply this uniformly without checking
who actually sees the error:**

| Subrepo | Who sees the final error | Needs `userMessage`? |
| --- | --- | --- |
| `server` (HTTP) | Another developer (the consuming app) — unless that app echoes `message` directly in its own UI | Yes, for business/validation `HttpError`s a consuming app is likely to render as-is (`CONFLICT`, business `NOT_FOUND`) |
| `cli` | The developer running the command — dev and end user at once | Not a priority — `message` is already the right audience |
| `space` (SSR error boundary) | A real end user, via `error.tsx`'s fallback UI | Yes — the boundary logs `error` but exposes nothing to the fallback; `userMessage` is the natural field for a human-readable render |
| `admin`/`asyncmq`/`datamaster`/`notifications` | Almost always another developer/operator — infrastructure libraries, no UI of their own | No, except real edge cases already identified per-package |
| `space-ui` | Depends on the consumer — the component doesn't decide audience | N/A — presentation layer, not domain; not this package's call to make |

## What actually reaches an HTTP client

`getPublicErrorResponse` (`server/src/utils/errors/helper.ts`) is a real
ALLOWLIST — `id`, `contextId`, `name`, `message`, `code`, `status`,
`userMessage`, plus `meta`/`cause` **only if the error explicitly opts in**
via `exposeMeta`/`exposeCause` (both default `false`). This is what
`getSerializedErrorResponse`/`httpErrorResponse` actually use — the only two
real paths to an external client.

`getExtendedErrorResponse` is the FULL, unrestricted version — `logAppError`
reuses it as-is to build what persists in the log, so `meta`/`cause` still
go to storage completely regardless of the HTTP allowlist (that's their
primary job). Don't confuse which one to call: allowlist for anything
reaching a client, extended for anything reaching a log.

**Two real, confirmed-legitimate cases needing `exposeMeta`/`exposeCause`**:
a protocol-version guard telling a client which version it declared vs. what's
supported (real client-facing `meta`); `@zanix/utils`'s own validation
`throwErrors` default, where per-field validation detail (`cause.properties`)
is feedback about the caller's own submitted data, not internal detail.

**`InternalError`/`ApplicationError`/`PermissionDenied`/an unwrapped native
`Error` reaching this layer now default to HTTP 500, not 400** — only
`HttpError` has an explicit `.status`; anything else reaching this point is
far more likely a genuine server-side failure than a client error. A
deliberate exception exists (GraphQL validation failures conventionally
return 400 per GraphQL-over-HTTP convention, not this same bug shape) — don't
"fix" a genuinely intentional 400 back to 500.

## Redaction: by key name, never by string content

The default pattern matches classic HTTP credential keys
(`authorization`/`cookie`/`password`/`token`/`secret`/`apiKey`/similar) plus a
PII/PCI batch added in this same audit (namespaced OTP fields, and similar
domain-specific additions — confirm the current `SENSITIVE_KEY_PATTERN` in
`@zanix/utils` before assuming a specific field is covered, don't rely on
memory of what this skill lists). **A secret under an unrecognized key name
passes through unredacted — this is a real, structural limitation, not an
edge case to dismiss.**

Extending the DEFAULT pattern (editing `SENSITIVE_KEY_PATTERN` in
`@zanix/utils`) is for something that applies to the WHOLE ecosystem equally
— confirmed real precedent: OTP fields were namespaced (`otpCode`/
`otpTarget`, not generic `code`/`target`) specifically to avoid mass false
positives, since `code`/`target` are common non-sensitive field names
elsewhere (an error's own `code`, for one). Something specific to one
domain goes via `redact.extend` at the call site, never the global default.

**A new sensitive `X-Znx-` header/cookie doesn't get redacted automatically
just by being in that namespace.** `SENSITIVE_KEY_PATTERN` matches by exact
key name, and today only special-cases two: `x-znx-authorization` and
`x-znx-app-token`. A new credential-carrying header a package introduces
(e.g. a service-to-service secret, a new machine-credential header) needs
one of these, deliberately, at the point it's introduced — it is not
covered by virtue of following the `X-Znx-` naming shape
(`naming-and-structure-conventions` owns that shape, not redaction):
- Whole-ecosystem: add it to `SENSITIVE_KEY_PATTERN` in `@zanix/utils`, same
  bar as the OTP precedent above (only when every consumer should redact it).
- One package/call site: `redact.extend` with the literal header/cookie
  name.
A header that only ever carries a non-secret value (an id, a status, a
protocol version — most of the existing `X-Znx-` catalog) needs neither.

## Sanitizing connector/driver errors

`sanitizeConnectionUri` (free-text strings — a driver error's `message`/
`stack` that can embed a full connection string with credentials) and
`sanitizeModel` (the Mongoose Model/Connection object itself) are
complementary, covering different leak surfaces — not redundant with each
other or with key-based redaction (a URI embedded in the MIDDLE of a message
string isn't under any recognizable key, redaction-by-key-name can't catch
it).

**Apply `sanitizeConnectionUri` at EVERY point a connector/driver error gets
logged, not just `initialize()`/`connect()`** — confirmed real gap, fixed:
`datamaster/mongo/connector/mod.ts` has it at BOTH `initialize()`'s catch
(~line 487) and the `close()`/disconnect catch (~line 534) — the second was
added specifically because the same raw driver error reaching the same
logger was missing it there, per the comment left in place ("Same class of
risk as `initialize()`'s own catch above ... not just at `initialize()`").
Same shape confirmed in `datamaster/cache/providers/redis/connector/mod.ts`
at two independent call sites too. Check every catch that logs a
connector/driver error, not just the obvious one.

## `enableALS`/correlation-id: still opt-in, don't assume otherwise

Stays a per-route flag, not a new default — there's a real, documented Deno
correctness blocker (`server/src/modules/infra/base/storage.ts`). Don't
enable it for a new project without checking that blocker still applies.

## Library vs. consumer — this skill applies differently on each side

- **`cli` and `space-ui` deliberately do NOT use the shared error hierarchy**
  — this is correct, not an exception to fix. `cli` has no HTTP layer (its
  "error" is dev-facing CLI UX); `space-ui` is a pure presentation library
  with no server-side `shouldLog`/redaction concerns. Don't flag either as
  non-conformant against the "always use the 4 shared classes" rule above.
- **A consumer project** (a `server`/`space`/`spacecraft` built with
  `zanix-feature-builder`, or a `zanix-remote-api-app-pattern`-shaped
  `space` app initially stood up by `zanix-remote-api-app-builder`) should
  follow: the shared error hierarchy, the
  `DOMAIN_SUBSYSTEM_REASON` `code` convention, the `'noSave'` rule, and
  `message`/`userMessage` per the audience table above.
- **A fix/feature inside an ecosystem library itself** (datamaster,
  notifications, auth, etc.) additionally must: never double-log a
  self-logging `InternalError`, and apply `sanitizeConnectionUri` at every
  connector/driver error log site, not just the obvious one — see both
  sections above for the real, confirmed precedents to match.

## Testing discipline this skill adds on top of `feature-completeness-conventions`

Confirmed real, twice, from this audit's own history: an early pass of error-
class fixes used `assertRejects(fn, Error, 'substring')`-shaped tests, which
pass identically against the OLD class too — they don't actually guard
against regression. Reinforced afterward to check the specific class
(`instanceof InternalError`/`ApplicationError`/etc.) + `.code`. Match the
strict form from the start:

- [ ] Does a new/changed thrown error get tested as `instanceof
      <SpecificClass>` + `.code`, never a generic `Error` + message
      substring?
- [ ] If the fix touches `shouldLog`, is `_logged` verified directly (same
      mechanism `processError` uses: `_logged = !!options.shouldLog`), not
      just that the error was thrown?
- [ ] If the fix is a class migration (native `Error`/`Deno.errors.*` →
      shared hierarchy), are EVERY caller's existing tests updated to the
      new class, not left passing coincidentally against the old one?

## Checklist before logging an event or throwing an error

- [ ] Persisting or `'noSave'`? Apply the rule above — is this the final
      outcome of an operation, or `high`/`error`/degradation-signaling
      `warn`? Then it persists. High-volume routine lifecycle or a
      non-terminal retry attempt? Then `'noSave'`.
- [ ] Right error class — did the CALLER cause this, or something outside
      their control? Check the table; never a bare `Error`/`Deno.errors.*`
      without a confirmed domain-specific justification (and even then,
      extend the shared hierarchy, don't replace it).
- [ ] `InternalError` with `shouldLog:true` — is there a redundant manual
      `logger.error` call right after constructing it? Remove it. This
      covers more than the obvious case: `_logged` stamped at construction
      also short-circuits a global uncaught-error handler (`logAppError`/
      `shouldNotLogError`) from re-logging the same instance if it
      propagates there — confirmed real for a boot-time throw reaching
      `self.onerror`, not just an adjacent manual call.
- [ ] Does `code` follow `DOMAIN_SUBSYSTEM_REASON`, and exist at all for
      anything crossing a public boundary — **except a boot-time/constructor
      error that can never reach a request/response cycle at all** (see the
      `code` section above); `meta`-only there is the real convention, not a
      gap. This exemption is about `code`/`userMessage` only — it does NOT
      also excuse the item above (still wrap in one of the four shared
      classes even at boot time; don't read "no `code` needed" as "no
      wrapping needed").
- [ ] `userMessage` — check the audience table; only set it where an error
      can realistically reach a real end user, never assume `message`
      itself is safe to show one. Same boot-time exemption as `code` — no
      request exists yet to attach a user-facing message to.
- [ ] Client-facing response — using `getPublicErrorResponse` (allowlist),
      not `getExtendedErrorResponse` (full, log-only)? `exposeMeta`/
      `exposeCause` only where genuinely client-relevant (protocol-version
      negotiation, per-field validation feedback), not by default.
- [ ] Any connector/driver error logged — is `sanitizeConnectionUri` applied
      at THIS specific call site, not assumed covered because it's applied
      elsewhere in the same file?
- [ ] Is this a `cli`/`space-ui` context, where the shared error hierarchy
      deliberately doesn't apply — don't force it there.
- [ ] Tests assert the specific class + `.code` (or `_logged` for
      `shouldLog` changes), not a generic `Error` + substring match.

## Out of scope — do not do these

- Logger/error-class internals — how `logger.high`'s color/icon/dedup work,
  `Logger`'s storage configuration styles, or the shared error classes'
  constructor mechanics. That's `utils-logger`/`utils-errors`'s job; this
  skill only covers which primitive to reach for and when.
- General test-coverage completeness — `feature-completeness-conventions`'s
  gate, already mandatory; this skill only adds the specific-class-not-
  substring assertion discipline on top of it.
- Whether a new env var needs a selector/guard shape — that's
  `zanix-envvar-conventions`'s job, unrelated to which log level or error
  class a piece of code should use.
- Building a centralized `code` registry to fix inconsistent naming —
  explicitly rejected above (see the `code` section); the convention is
  mandatory, a shared catalog is not the fix.
- Deciding whether a log/error decision is itself documented in
  README/JSDoc — that's `feature-completeness-conventions`/
  `docs-readme-audit`'s gate; this skill only judges the decision's
  correctness, not its documentation coverage.

---
name: naming-and-structure-conventions
description: Folder/file casing, test-file naming, constant-vs-behavior casing (camelCase/UPPER_SNAKE_CASE/PascalCase), the env-var _ENV constant-naming shape, and the `X-Znx-` HTTP header/cookie naming shape (including the hard `cookiesGuard` prefix constraint) — the naming/structure rules that apply uniformly across every Zanix repo, independent of any one package's architecture. Use whenever creating or renaming a file/folder, naming a constant or exported symbol, naming a test, or naming a new HTTP header/cookie. Does NOT own env-var SELECTOR shape (pattern A/B/C/D, X_MODE resolvers) — that's zanix-envvar-conventions; this skill only owns how the constant that holds an env var's name is itself named. Does NOT own whether a new header/cookie needs redaction — see zanix-observability-conventions for that.
---

Grounded in a full 12-repo audit (2026-08-19/20, "Zanix Naming Audit") — 63
real violations found across ~2,300+ files, zero high-severity, 12/12 repos
independently confirming `@tests/` (not `@test/`). Re-verify a specific
repo's current state before citing it as fully conformant; this skill
records the standard and the known open gaps, not a permanent snapshot —
several of the findings below are still unfixed as of this writing.

## Golden rule (token savings)

- **The rules below are almost all already how this ecosystem writes code.**
  This audit exists to make the *document* match reality, not to retrain
  anyone — treat a violation as a targeted fix, not evidence the whole area
  needs a naming pass.
- **This skill doesn't own env-var SELECTOR shape.** Whether a group of env
  vars needs an `X_MODE` selector is `zanix-envvar-conventions`'s
  classification (pattern A/B/C/D). This skill only owns how the constant
  holding one var's literal name is itself named — rule 5 below.
- **Don't invent a 7th exception.** Every genuine edge case this audit found
  (RegExp/Date literals, const-enum containers, `import.meta.url` identity
  refs, decorator-invoked factories) is already resolved in rule 4. Match
  one of those, don't reason a new one into existence for a case that isn't
  actually different in shape.

## 1 · Folders

`kebab-case`. Confirmed exceptions:

- `@tests/` — not `@test/`. Confirmed without exception across all 12 repos,
  including explicit cross-checks from 3 of them.
- `__tmp__/`, and by extension any test-infrastructure folder/file wrapped
  in double underscores (`__fixtures__/`, `__setup__.ts`).
- React/TSX component folders in PascalCase matching the component name.
- Names required by a framework, runtime, or OS (never renamed to fit this
  rule).

## 2 · Files

`kebab-case`. Confirmed exceptions:

- React/TSX components in PascalCase (in `space-ui`, the real entrypoint is
  usually `index.ts`, not a `.tsx` file — the exception applies to the
  *folder*, not always to a file inside it).
- Fixed tooling names (`mod.ts`, `deno.jsonc`, `*.config.ts`).

**Resolved, human-approved 2026-08-20.** `docs/*.md` casing was split at the
original audit (8/12 repos lowercase kebab-case, 4/12
`SCREAMING-KEBAB-CASE`) — the recommendation to adopt the majority
(lowercase) as the single ecosystem-wide standard is now approved and
executed: `app`, `datamaster`, `server`, and `cli` had every `docs/*.md`
file renamed to lowercase kebab-case, cross-references updated in the same
change (`ENGINEERING.md`→`engineering.md`, `DEPLOY.md`→`deploy.md`, etc.).
As of this writing the rename is real on disk but not yet committed in
those 4 repos — re-verify before citing a specific file as landed if this
becomes stale. `docs/*.md` is now lowercase kebab-case, no exception,
ecosystem-wide — same as every other file this rule covers.

## 3 · Tests

`.test.ts`/`.test.tsx` suffix — universal across all 12 repos, zero mixing
with `.spec.ts`. The base name must equal the module under test **only for
true 1:1 unit tests**. Exempt from that base-name match:

- Functional/integration tests — these test a scenario, not a file, by
  design.
- A unit test that covers one cross-cutting responsibility spread across
  several modules (e.g. a decorator-wrapper test, a cross-module validation
  test) — naming it after any single module it touches would be arbitrary.
- `cli`'s formal three-tier `unit`/`integration`/`functional` suite — richer
  than the binary model this rule otherwise assumes, already codified in
  that repo's own `deno.jsonc` and documented in its own skill. Treat it as
  a sanctioned variant of `@tests/`'s structure, not an exception to flag.

## 4 · Constants vs. values with behavior

| Casing | Applies to |
| --- | --- |
| `camelCase` | Functions, factories, exported instances/APIs — even when declared `export const`. **Having callable methods isn't sufficient on its own** — check the `RegExp`/`Date` edge case below before applying this row to anything method-bearing; confirmed real miss: a `RegExp` literal (`.test()`/`.exec()`) reads as an "instance/API" from this row alone, but is resolved the OPPOSITE way below. |
| `UPPER_SNAKE_CASE` | Conceptual constants with no behavior, **including objects/Records/arrays holding static config** — not just scalars. |
| `PascalCase` | Classes, interfaces, types, enums, components, **and factories invoked as a decorator** (`@Controller`, `@Get`, `@Page`, `IsString`, `IsEmail`) — these are called as a declarative annotation, not an ordinary function call. Confirmed as a real, recurring pattern in 4 repos, not an isolated slip. |

Edge cases this audit resolved — match one of these, don't re-derive:

- **Static `RegExp`/`Date` literals** with no mutable state of their own →
  conceptual constant, `UPPER_SNAKE_CASE`, even though they expose
  prototype methods. Confirmed real miss, not hypothetical: `@zanix/utils`'s
  own `src/utils/regex.ts` had ~17 regex constants in camelCase
  (`emailRegex`, `uuidRegex`, `objectIdRegex`, ...) — a whole-file,
  pre-existing violation of this exact rule, only caught when a `cli`
  generator's own `OBJECTID_REGEX` (correctly `UPPER_SNAKE_CASE`) was
  mistaken for the naming violation instead, before this rule was found.
  Since fixed in `utils` (all renamed to `UPPER_SNAKE_CASE`, e.g.
  `OBJECT_ID_REGEX`).
- **A const-enum-shaped container object** (paired with a derived type like
  `typeof X[keyof typeof X]`) → `UPPER_SNAKE_CASE` on the container, same as
  its members.
- **Runtime-computed identity references** (`import.meta.url` assigned to a
  constant to instantiate a `Worker`) → `camelCase` — these are derived at
  runtime, not authored config, even though they look like static values.
- **Acronyms inside PascalCase symbols** (`DLQ`) → one casing per repo, not
  per file. Don't mix `DLQAggregator` and `DlqControllerOptions` for the
  same acronym, even in the same file.
- **A static CSS-in-JS style-object constant** (a `Record`/object literal
  meant to be passed straight into a `style` prop) → `UPPER_SNAKE_CASE` on
  the constant itself, same as any other static config object — easy to
  second-guess, since CSS-in-JS conventionally uses camelCase *keys*
  (mirroring DOM style property names like `backgroundColor`) inside the
  object. That convention governs the keys, not the constant's own name.
  `space-ui`'s own `DRAWER_SIDE_STYLE`/`MODAL_Z_INDEX` are the real,
  confirmed precedent for this exact shape (`visuallyHiddenStyle`, in the
  same package, was the one place this wasn't followed — since fixed to
  `VISUALLY_HIDDEN_STYLE`, see "Fixed since the original audit" below).

## 5 · Env var constants

`UPPER_SNAKE_CASE`. Every `Deno.env.get/set/has('X')` read must be backed by
an exported `X_ENV` constant. **The naming direction matters and is easy to
invert** — verified against 20+ consistent instances in `datamaster` and
cross-confirmed by `zanix-envvar-conventions`'s own real precedents
(`SEARCH_ENGINE_ENV`, `TEMPLATES_BACKEND_ENV`): the constant holds the
env var's *literal name*, resolved at the point of use — not its resolved
value.

```ts
// Wrong — the constant IS the resolved value, not the var's name
const MONGO_URI_ENV = Deno.env.get('MONGO_URI')

// Right — the constant holds the literal name; resolve at the use site
const MONGO_URI_ENV = 'MONGO_URI'
// ...
const uri = Deno.env.get(MONGO_URI_ENV)
```

This is orthogonal to whether the var needs a selector at all — see
`zanix-envvar-conventions` for that classification (pattern A/B/C/D). A
typo'd inline literal fails silently at runtime; a typo'd constant reference
fails at compile time — export the constant regardless of the var's
pattern, on every new var.

## 6 · HTTP headers and cookies — the `X-Znx-` namespace

Every framework-owned HTTP header/cookie is `X-Znx-<Word>-<Word>-...`
(PascalCase words, hyphen-separated) — confirmed consistent across `server`,
`auth`, `admin`, `space`, and `notifications`: `X-Znx-Authorization`,
`X-Znx-Admin-Protocol`, `X-Znx-RateLimit-Limit`, `X-Znx-User-Session-Status`,
`X-Znx-Discovery-Protocol`, `X-Znx-Csrf`, `X-Znx-Lang`, `X-Znx-Population`.
(`asyncmq`'s own `x-znx-*` AMQP message headers are a separate namespace —
message metadata, not HTTP — same prefix, unrelated mechanism.)

**Hard constraint, not just a naming preference, for cookies specifically**:
`@zanix/server`'s `cookiesGuard` filters `ctx.cookies` down to only
`X-Znx-`-prefixed names before any guard runs. A cookie set outside that
prefix (`znx-csrf`, `session-id`, ...) is issued and echoed fine at the raw
HTTP level but is **silently invisible** to every guard/handler reading
`ctx.cookies` — there's no error, no warning, just a value nothing in the
framework can ever see. Any new cookie a package/app sets must start with
`X-Znx-`, full stop.

Naming a *new* header that carries a credential/secret is a different
question from naming it correctly — see `zanix-observability-conventions`
for when it also needs registering with the redaction layer
(`SENSITIVE_KEY_PATTERN`/`redact.extend`) so it doesn't leak into logs.

**Deciding whether a new header/cookie's name should be a fixed constant or
a customizable option** — the real, confirmed split in today's code:

- **Fixed constant, never customizable** — when the name is a contract
  BETWEEN independently-versioned packages/services that must agree on the
  literal string without coordinating at runtime. Confirmed real precedent:
  `AUTH_HEADERS`/`SESSION_HEADERS`/`GENERAL_HEADERS`/`ADMIN_PROTOCOL_HEADER`/
  `PROTOCOL_VERSION_HEADER`/`DISCOVERY_PROTOCOL_HEADER` all live in
  `@zanix/server` specifically so `@zanix/auth`/`@zanix/admin`/
  `@zanix/notifications` import the SAME constant instead of each hardcoding
  its own copy — `notifications`'s `RemoteTemplateBackend` used to hardcode
  its own local copies of `X-Znx-Admin-Protocol`/`X-Znx-Authorization` and
  was fixed to import `ADMIN_PROTOCOL_HEADER`/`AUTH_HEADERS` instead (see
  its CHANGELOG). If one of these were customizable per-app, the *other*
  service reading it wouldn't know what name to look for.
- **Customizable via an option (e.g. `cookieName`)** — safe only when the
  exact same guard/mechanism both SETS and READS the value within one app's
  own request lifecycle, with no other package depending on the literal
  name. Confirmed real precedent: `csrfGuard`'s (`X-Znx-Csrf`),
  `langGuard`+`langPreHandler`'s (`X-Znx-Lang`, must match between the two),
  and `populationGuard`'s (`X-Znx-Population`) cookie names are all
  customizable — and confirmed, none of the three appear anywhere outside
  `@zanix/space` itself.

Either way, a customized name must still satisfy the base `X-Znx-` prefix
shape above — there's no runtime or lint enforcement of that today (honor
system via JSDoc only), a known gap tracked as an issue rather than grown
here as skill prose.

## 7 · Cross-OS casing collisions

Zero found across all 12 repos as of this audit (macOS case-insensitive vs.
Linux case-sensitive divergence) — the ecosystem starts from a clean base
here. Generated/vendored code is excluded consistently everywhere this was
checked. Worth a quick check on any new file whose name differs from a
sibling only by casing, but not a standing concern otherwise.

## Confirmed exceptions, not violations

- **`middlewares/*.guard.ts`/`*.pipe.ts`/`*.interceptor.ts` (suffix-in-file,
  raw implementation) vs. `middlewares/decorators/*.ts` (subfolder, no
  suffix, PascalCase decorator-factory wrapping the implementation) are
  two genuinely distinct artifact kinds, not two names for the same
  thing** — confirmed in `auth` and matching the identical pattern in
  `server`, `asyncmq`, and `utils`. Don't flag this pairing as a casing
  inconsistency; it's the repo-wide convention for "raw middleware" vs.
  "its `@Name(...)` decorator."
- **A test named after the specific behavior/symbol it exercises, not its
  module's basename, is exempt from rule 3 when the name is itself
  grep-discoverable** (prefixed with the module name + a scenario, or
  matching an exact exported symbol) — e.g. `server`'s
  `database-name-empty.test.ts`/`database-name-truncate.test.ts` (both
  test `connectors/core/database.ts`), `graphql-depth-limit.test.ts`
  (tests `graphql/handler.ts`'s `createDepthLimitRule`), and
  `resolve-global-prefix.test.ts` (tests `webserver/mod.ts`'s
  `resolveGlobalPrefix`). Narrower than the functional/integration
  exemption — applies only when the name still makes the module
  findable, not to a vague scenario name with no traceable anchor.

## Fixed since the original audit — cite these as worked precedent, not as an open gap

- **Test base-name mismatches — all resolved.** `server` (9 renames + 1
  redistribution: `utils.test.ts`'s 3 unrelated `Deno.test` blocks moved
  into their real homes, `contextId`→`utils/context.test.ts`,
  `pathToRegex`→`utils/routes.test.ts`, `getTargetKey`→
  `utils/targets.test.ts` — this surfaced a real bug: the relocated
  `getTargetKey` test had hardcoded absolute counter values that broke
  once it shared a module-level counter with its new file's other tests;
  fixed by asserting the real contract, not a magic number), `utils` (10
  relocations to `unit/utils/` + `strings.test.ts` merged into
  `encoders.test.ts`, since it wasn't an orphan — it covered 8 real
  `encoders.ts` functions `encoders.test.ts` didn't touch, just under the
  wrong name), `cli` (1), `asyncmq` (2: `prepare.test.ts` renamed to
  `rabbitmq-messages.test.ts` following the repo's own
  `rabbitmq-*.test.ts` precedent, `jobs.test.ts` split into
  `cron.test.ts`/`task.test.ts` since the two cases shared no fixture and
  tested independent modules), `auth` (5 — the `middlewares/` structure
  "blocker" turned out not to be one at all, see "Confirmed exceptions"
  above; `scopes.test.ts`→`scope.test.ts` had been miscategorized as
  depending on that decision when it doesn't even test `middlewares/`),
  `app` (3 of 4 — `serve.test.ts`→`define.test.ts`,
  `graph-and-validate.test.ts` split into `graph.test.ts`/`validate.test.ts`
  since the source modules document them as separate responsibilities,
  `http-remote-adapter-tls.test.ts`→`http-remote-adapter.test.ts`
  following the repo's own same-basename-different-tier pattern; the 4th,
  `target-identity.test.ts`, was confirmed genuinely exempt — it records a
  `@zanix/server` guarantee, not behavior owned by any module in `app`).
  A second pass on `app` found the original audit's own premise wrong for
  3 of these 4 (it claimed each had an explaining JSDoc; only one did) —
  don't inherit a prior audit's "looks deliberate" conclusion without
  reading the actual comment yourself.
- `datamaster` — the 7 inline env vars (`MONGO_URI`, `REDIS_URI`,
  `MEMCACHED_URI`, `LOCAL_CACHE_MAX_ITEMS`, `ELASTICSEARCH_API_KEY`,
  `OPENSEARCH_API_KEY`, `AMQP_URI`) now each have an exported `_ENV`
  constant, resolved via `Deno.env.get(X_ENV)` like the rest of the repo.
  This was the one finding in the whole audit with real functional risk
  (a typo'd inline literal fails silently) — everything else was cosmetic.
- `datamaster`, the two follow-up findings `conventions-sweep`'s first real
  validation run surfaced (not covered by the 7-var entry directly above,
  since those 7 already had a constant defined elsewhere — this was about
  call sites/vars that had none at all) — both now fixed. (1)
  `database/providers/mongo/connector/core.ts:31`, `cache/providers/redis/core.ts:26`,
  `cache/providers/memcached/core.ts:26` now call
  `Deno.env.has(MONGO_URI_ENV|REDIS_URI_ENV|MEMCACHED_URI_ENV)` instead of
  the raw literal, matching `storage/core.ts`'s
  `Deno.env.has(S3_ENDPOINT_ENV)` precedent. (2) `src/utils/protection.ts`'s
  `DATA_SECRET_KEY`/`DATA_AES_KEY`/`DATA_RSA_PUB`/`DATA_RSA_KEY` family now
  each have an exported `_ENV` constant (`DATA_SECRET_KEY_ENV`/
  `DATA_AES_KEY_ENV`/`DATA_RSA_PUB_ENV`/`DATA_RSA_KEY_ENV`), resolved at
  each use site including the `_V1`/`_V2`/... versioned-suffix
  concatenation. The same file's `getMaskSecret`/`getEncryptSecret` also
  now throw `InternalError` (codes `DATAMASTER_MASK_KEY_MISSING`/
  `DATAMASTER_ENCRYPTION_KEY_MISSING`, `shouldLog: false` since the
  surrounding `mask`/`unmask`/`encrypt`/`decrypt` catch already logs it
  under its own code — constructing with the class default
  `shouldLog: true` here would have double-logged) instead of a native
  `Error`, matching this file's own sibling catch blocks and
  `storage/encryption.ts`'s `requireEnv`. **Neither fix touched a public
  export**: `getMaskSecret`/`getEncryptSecret` are module-private, and the
  `InternalError` they throw never escapes the try/catch inside `mask`/
  `unmask`/`encrypt`/`decrypt` (all four ARE exported, re-exported from
  `mod.ts` as `datamasterMask`/`datamasterUnmask`/`datamasterEncrypt`/
  `datamasterDecrypt` — checked directly, per the `space-ui`/`admin` lesson
  below, before concluding this) — every one of them still catches,
  logs once, and returns the original input unchanged, exactly as before.
  Not `BREAKING`. Covered by a new `src/@tests/unit/utils/protection.test.ts`
  (asserts the `_ENV` constants' literal values, and that each of the 4
  public functions logs exactly once — not twice — under the pre-existing
  code when its key is missing).
- `server` — `ConnectorCoreModules`/`ProviderCoreModules`
  (`modules/infra/{connectors,providers}/core/all.ts`) renamed to
  `connectorCoreModules`/`providerCoreModules`. Confirmed no external
  consumers, zero-risk rename, ~34 internal sites updated.
- `space` — `@tests/benchmarks/space/unused/**` (12 unhyphenated files in
  an already-orphaned directory) deleted entirely, along with the empty
  `src/shared/`.
- `notifications` — `smtpResponseCode` renamed to `SMTP_RESPONSE_CODE`
  (const-enum-shaped container, paired with a derived
  `typeof SMTP_RESPONSE_CODE[keyof ...]` type). Not a public export,
  zero-risk.
- `asyncmq` — `deadletterOpts`/`schedulerOpts` renamed to
  `DEADLETTER_OPTS`/`SCHEDULER_OPTS`. Not a public export, zero-risk.
- `cli` — `linterBaseRules`/`editors`/`defaultNpmModules` renamed to
  `LINTER_BASE_RULES`/`EDITORS`/`DEFAULT_NPM_MODULES`. Not public exports,
  zero-risk.
- `space-ui` — `visuallyHiddenStyle` renamed to `VISUALLY_HIDDEN_STYLE`.
  **This one WAS a public export** (`mod.ts`/`mod-preact.ts`) despite the
  original audit assuming otherwise — a genuine breaking change, shipped
  with a CHANGELOG entry marked `BREAKING`, version bump still pending
  the next release. Don't trust an audit's "not publicly exported" claim
  without grepping `mod.ts`/`deno.jsonc`'s `exports` yourself first.
- `admin` — `DLQAggregator`/`DLQAdminClient` renamed to
  `DlqAggregator`/`DlqAdminClient`, converging the repo on `Dlq` as the
  one casing for the `DLQ` acronym. **Also a public export** — same
  breaking-change treatment as `space-ui` above, same lesson: verify
  export status yourself, don't inherit an audit's assumption.
- `datamaster` — same `DLQ`→`Dlq` casing convergence as `admin` above
  (`DLQProvider`→`DlqProvider`, `DLQAdminService`→`DlqAdminService`,
  `registerDLQModel`→`registerDlqModel`, and 21 more — full list in
  `mod.ts`/`src/modules/dlq/core.ts`/`src/modules/dlq/dlq-api/mod.ts`), but a
  **deliberately different shipping decision for the same kind of change**:
  every old name is kept as an `@deprecated` re-export of the exact same
  binding (`export { DlqProvider as DLQProvider } from '...'`, `@deprecated
  Use {@link DlqProvider} instead` JSDoc, grouped under a "Deprecated
  aliases" comment block in each of those three files), so this ships as
  `MINOR`, not `MAJOR` — unlike `admin`'s shim-less hard rename. Don't read
  this as `admin` having done it "wrong": the two calls were made for
  different reasons — `admin`'s blast radius and consumer count justified a
  clean break, while `datamaster`'s real blast radius today is small and
  known (`admin`/`core` as the only consumers, `admin` already shielded by
  its own `^1.5.0` range either way), so its own major bump is deliberately
  saved for when more casing cleanup of this kind accumulates rather than
  spent on this alone. `Dlq`-cased names are still the one recommended form
  going forward in new code either way — the alias exists for compatibility
  during the deprecation window, not as a second sanctioned casing.

## Checklist before naming a new file/folder/constant

- [ ] Folder/file kebab-case, or a confirmed exception from rules 1–2 (not a
      new one reasoned into existence)?
- [ ] Test file ends in `.test.ts`/`.test.tsx`, and its base name matches
      its module — unless it's a functional/integration test or a
      genuinely cross-cutting unit test (rule 3)?
- [ ] Every exported symbol cased by what it *is* (behavior → camelCase,
      static data even if an object/array → UPPER_SNAKE_CASE, class/type/
      decorator-factory → PascalCase) — not by what it looks like it should
      be from habit?
- [ ] Every new `Deno.env.get/set/has('X')` backed by an exported `X_ENV`
      constant holding the literal name, not the resolved value (rule 5)?
- [ ] Any new HTTP header/cookie named `X-Znx-<PascalCase>-...`, and — if
      it's a cookie — is `X-Znx-` actually the prefix, not just somewhere in
      the name (rule 6, or `cookiesGuard` hides it silently)?
- [ ] Any item in "Known current gaps" above not touched as an unrelated
      side effect of this change?
- [ ] A new `docs/*.md` file named lowercase kebab-case (rule 2's resolved
      standard), not `SCREAMING-KEBAB-CASE`?

## Out of scope — do not do these

- Re-opening the `docs/*.md` casing decision — it's resolved (rule 2);
  don't relitigate it because a specific file feels like an exception.
- Env-var SELECTOR shape (whether a group needs an `X_MODE` resolver) —
  that's `zanix-envvar-conventions`'s classification, not this skill's.
- Bulk-renaming an entire repo's existing violations as a side effect of an
  unrelated change — fix the file(s) the current task actually touches;
  note the rest as a follow-up, don't sweep unrelated files into scope.
- Deciding remediation order across multiple findings — report what's
  non-conformant and whether it carries real risk (like the `datamaster`
  `_ENV` gap does); sequencing a whole-ecosystem cleanup is a human call.

---
name: deno-lazy-dependency-pattern
description: How to declare a genuinely conditional/optional dependency in a published Deno/JSR package without `nodeModulesDir: "auto"` eagerly materializing its npm packages for every consumer, whether they use the capability or not — the `lazyFunction`/`lazyClass`/`lazyValue` helpers (`@zanix/utils/helpers`), the real `import type` gotcha, a `.tsx` file's own extension-driven `jsx-runtime` injection independent of real JSX content, and when to fix the SOURCE package's own export/import shape instead. Applies to any Deno/JSR package, inside or outside this ecosystem — not Zanix-architecture-specific. Use when a package needs to depend on something only some consumers/manifests actually use, or when investigating why `zanix space build`/`deno check` downloads npm packages (including `react`/`preact`) a project never imports.
---

Grounded in a real, same-day investigation (2026-08-25) that started from one observation —
`zanix space build` on a plain `space` project with no `@zanix/core`/`@zanix/datamaster`
dependency still downloaded `mongoose`/`redis`/`amqplib` — and ended up tracing the exact
mechanism through `@zanix/app`, `@zanix/asyncmq`, `@zanix/utils`, and `@zanix/server` itself, with
every claim below verified via a real, isolated `deno check`/`node_modules` repro, not assumed.
Two independent sessions (working on `@zanix/app` and `@zanix/utils` respectively) converged on
the same root cause and cross-validated each other's fixes. **A dedicated validation pass against
`@zanix/datamaster` then found this skill's own first draft overclaimed one mechanism and missed
another** (both folded in below, not hidden) — treat that as the standard this skill holds itself
to, not a one-time correction: verify a claim from here against a real repro before trusting it on
a new package, the same way the validation pass did.

## Golden rule (token savings)

- **The bug is `nodeModulesDir: "auto"`, not any one package.** Confirmed via the official Deno
  docs and a live repro: with `"auto"` set, Deno materializes — npm-install style — EVERY npm
  package reachable from a `deno.json`'s `imports` map, regardless of whether any code actually
  imports it. A bare alias sitting in `imports`, unused, is on its own enough to trigger this.
  `"auto"` is also the only setting compatible with a real Vite/Rolldown pipeline — confirmed by
  breaking it on purpose (`"none"` fails a real `zanix space build` two different ways: a real
  npm package's install lifecycle script gets silently skipped, and Rolldown's own worker-bundling
  path throws a hard parse error). **Don't recommend changing `nodeModulesDir` as the fix** — it's
  also workspace-root-only (a dependency's own `nodeModulesDir` setting is provably invisible to
  its consumers, per Deno's own docs), so changing it in the SOURCE package wouldn't even reach
  the problem.
- **What actually triggers materialization is REACHABILITY of a real source file's own import
  graph — not merely whether a specifier is declared somewhere in a package's `imports` map.**
  This is the load-bearing mental model everything else in this skill follows from, and the one
  the first draft got wrong (see the two corrected rules right below). A JSR package publishes
  real TypeScript SOURCE, not compiled `.d.ts` — resolving ANYTHING from an entry file (a value
  import, or a type that has to be inferred from that file's own code) requires Deno to load and
  resolve that ENTRY FILE's own top-level imports, in full, exactly like any ES module — this is
  ordinary module semantics, not a Deno-specific quirk. `nodeModulesDir: "auto"` then
  npm-install-materializes every `npm:` package that graph touches, whether or not the specific
  symbol you asked for actually uses it. **A package's shared `imports` map having an entry for a
  heavy dependency is not, by itself, evidence that importing some OTHER unrelated subpath will
  materialize it** — confirmed by direct reproduction: `@zanix/datamaster`'s `deno.jsonc` declares
  `mongoose`/`redis`/`@aws-sdk/client-s3` at the top level (for `/database`/`/storage`'s sake), but
  a consumer importing only `@zanix/datamaster/cache` materializes neither `mongoose` nor
  `@aws-sdk/client-s3` — because `/cache`'s own entry file (`cache/mod.ts`) never itself imports
  anything that reaches them. The two real triggers, confirmed distinct:
  1. **An unused `npm:`-literal alias sitting in the CURRENT project's own `imports` map**
     (the one actually being built/checked, not a resolved dependency) IS npm-install-style
     materialized regardless of reachability — this is closer to how `package.json`
     `dependencies` behave than to ordinary ES module resolution. Confirmed by direct reproduction
     (a bare `"mongoose": "npm:mongoose@^8"` entry, never imported anywhere, still materializes).
  2. **An entry in a DEPENDENCY's (an already-published package's) own `imports` map** is NOT
     eagerly pulled just by existing — only if some file in the ACTUALLY-REACHED part of that
     dependency's own source (the specific entry file/subpath a consumer imports, and whatever
     it in turn imports) resolves that specifier itself. This is why a genuinely narrow subpath
     (see the next rule) really can be sufficient on its own.
- **`import type` is NOT automatically safe — the rule everyone gets wrong first, but check
  REACHABILITY of the specific entry point, not just "does this package have npm deps somewhere."**
  Confirmed by direct, repeated reproduction: `import type { Job } from 'jsr:@zanix/asyncmq@^0.7.0'`
  materializes `amqplib`/`graphql`/`redis`, because `asyncmq`'s bare `.` entry file (`mod.ts`) is a
  single-file barrel that re-exports the RabbitMQ connector, providers, AND job registration all
  together — so resolving `Job`'s type through that entry point forces the whole file's own import
  graph to resolve, connector included. **The precondition check is two-part, not one grep of a
  manifest**: (1) trace which entry file the specifier actually resolves to (the bare `.`, or a
  narrower subpath if one exists and is genuinely being imported), and (2) check THAT file's own
  real import graph (grep the actual source, not just `deno.jsonc` — see the inline-literal-
  specifier gotcha below, which a manifest-only grep misses entirely) for anything npm-backed,
  directly or transitively. A source package with real npm dependencies SOMEWHERE in it is not
  automatically unsafe to `import type` from — only unsafe if the SPECIFIC entry point you're
  resolving reaches them.
- **A genuinely narrow `exports` subpath CAN be sufficient on its own — verify with a real repro
  before assuming it isn't.** The first draft of this skill claimed a narrow subpath alone never
  fixes this; that's wrong, and empirically disproven by the `@zanix/datamaster/cache` case above.
  What actually determines it: does THIS subpath's own entry file's real reachable graph (value
  and type edges both) touch the heavy dependency? If no file in that graph does, the subpath is
  safe as a normal, static, non-lazy import — full stop, no lazy machinery needed. If it DOES
  (confirmed real case: `@zanix/asyncmq`'s bare `.` bundling the RabbitMQ connector with job
  registration in one file), fixing it needs BOTH a narrower subpath (splits which SYMBOLS a
  consumer gets, onto a leaner entry file) AND, only if that narrower entry file's own graph still
  touches something heavy, removing that from the package's top-level `imports` and resolving it
  via this same lazy pattern internally. Don't assume the second half is needed without checking
  the narrowed entry file's own graph first — the `@zanix/datamaster` case shows it often isn't.
- **This recurs at every layer of a dependency chain — check the whole chain, not just the one
  package you're touching.** Real confirmed case: fixing `@zanix/app`'s own over-broad `imports`
  wasn't the end of the story — `@zanix/app` depends on `@zanix/asyncmq`, whose own `mod.ts`
  barrel bundles job registration with its RabbitMQ connector, AND whose own `deno.jsonc`
  separately declares `@zanix/datamaster` at the top level (for its DLQ integration) — so even a
  fully-lazy `app`-side fix still materializes `mongoose`/`redis` transitively through `asyncmq`,
  which itself needs the identical treatment for `@zanix/datamaster`. Trace the chain to its real
  end before declaring a fix complete.

## Before picking a technique — confirm the dependency is even in scope

Not every `npm:`/`jsr:` specifier that looks "conditional" needs any of this. Three real,
distinct categories, confirmed via today's sweep across `@zanix/admin`/`@zanix/core`/`@zanix/cli`/
`@zanix/space`/`@zanix/notifications` (2026-08-25) — sort a dependency into one before reaching for
any fix:

1. **Core, always-needed by the package itself.** A plain alias in that package's own `deno.jsonc`
   `imports` map already centralizes it for free — every internal file importing the bare alias
   name gets the one declared version, and bumping it is a one-line change in that one map entry.
   Confirmed real: `@zanix/notifications`'s own `handlebars`/`zod` (its `TemplateProvider` always
   needs them) need nothing beyond this. **Don't invent a `specifiers.ts` entry for these** — that
   file exists for a different problem (below), not as a blanket "all npm versions live here" rule.
2. **Conditional, reachable from a published `exports` entry.** This is what the rest of this skill
   describes — a consumer of the package might never trigger the capability, so the dependency must
   stay out of both the `imports` map and any literal specifier, per the pattern below.
3. **Build-tooling only, never reachable from any published `exports` entry.** Confirmed real:
   `@zanix/notifications`'s own `src/modules/templates/handlebars/compiler.ts` (a `deno task
   build-handlebars`-only script generating compiled templates ahead of time) imports `gulp`/
   `gulp-handlebars`/`gulp-rename`/`gulp-wrap` as literal `npm:` specifiers, and NOTHING else in the
   package imports this file — not `mod.ts`, not any subpath, nothing (verified by grepping every
   `.ts` file in the repo for the file's own path; the only two hits were doc comments, not real
   imports). No consumer's `deno info`/build ever reaches it, so there is no reachability edge to
   protect — this is a standalone build tool, the same category as a `vite.config.ts` a package
   might also ship. **Don't lazify this and don't give it a `specifiers.ts` entry**: wrapping an
   always-immediately-needed import in `await import()` adds ceremony with zero benefit (the script
   still needs the module the instant it runs, every time), and a `specifiers.ts` constant only
   earns its keep when the SAME literal is duplicated across 2+ files (see the file-not-found case
   below for why keeping the literal inline, in the one file that has it, is correct here). The one
   real reason to keep it OUT of the package's own top-level `imports` map even though no external
   consumer is ever affected: an UNUSED bare alias there still gets npm-install-materialized for the
   package's OWN local `deno check`/`deno test` runs (golden rule bullet 1), needlessly slowing down
   that package's own dev loop and CI — a real, if narrower, cost worth avoiding even when consumers
   are never at risk.

## Which technique — decide before reaching for `lazyFunction`/`lazyClass`

Three real, distinct techniques fix this, and they are NOT interchangeable — using the wrong one
either doesn't fix the leak or adds unnecessary async/complexity where a plain, synchronous fix
would do. Decide with this test, in order:

1. **Does the narrowed entry file's own reachable graph (value AND type edges) simply never touch
   the heavy dependency?** If yes — a plain narrow `exports` subpath is sufficient on its own, full
   stop. No runtime mechanism needed at all (see the golden rule above, `@zanix/datamaster/cache`'s
   real case).
2. **If it does touch it: does something else in the SAME package already, unconditionally, import
   the heavy file as part of doing its own genuinely-necessary job — with no runtime
   config/manifest check involved in deciding whether to import it at all?** If yes — use a
   **self-registration slot**, not `lazyFunction`/`lazyClass`. Confirmed real case: `@zanix/server`'s
   `getMainHandler` (shared by REST/Socket/SSR/GraphQL, needs the GraphQL handler SYNCHRONOUSLY —
   `WebServerManager.create()` returns a `ServerID` directly, not a `Promise`, so `await import()`
   was never viable here). Fix: a plain, dependency-free registration slot
   (`handlers/graphql/registry.ts` — `registerGraphqlHandlerFactory`/`getGraphqlHandlerFactory`).
   The real, heavy file (`handlers/graphql/handler.ts`) registers itself into the slot as a
   top-level side effect — which already happens automatically the moment ANYTHING reaches
   `@zanix/server/graphql`'s decorators (`Resolver`/`Query`/`Mutation`/`Request` all transitively
   import `handler.ts` for `getRootValueBucket`/`RequestContext` — using those decorators at all
   IS the trigger, no separate "is this feature enabled" check needed). The shared file reads the
   slot instead of statically importing the heavy file — zero async introduced, zero behavior
   change for an existing consumer, and the shared file's own module graph never has an edge to
   the heavy one. The trade-off: if NOTHING in the whole app ever reached `@zanix/server/graphql`,
   the slot stays empty and building a `'graphql'` server throws a clear error instead of silently
   building an empty stub — confirmed no real consumer relied on the old silent-stub behavior
   before shipping this.
3. **If neither applies — the dependency is genuinely EXTERNAL (a different package) and whether
   to use it at all can only be decided from RUNTIME state (a manifest, a config flag), with no
   other code path that would naturally trigger loading it first** — that's when `lazyFunction`/
   `lazyClass`/`lazyValue` earn their keep. Confirmed real case: `@zanix/app`'s `activateApps()`
   runs for every Zanix App, including a plain frontend `space` manifest that never declares
   `jobs`/`resources`/`operations` — nothing else in `app` ever imports `@zanix/asyncmq`/
   `@zanix/datamaster`/`@zanix/auth` on its own, so there is no self-registration trigger to piggy-
   back on. The only way to know whether to load them is inspecting the normalized manifest at
   runtime, which genuinely requires a conditional `await import()` — see the full pattern below.

4. **The package IS a CLI tool (a command-dispatch tree), and the heavy dependency belongs to one
   specific subcommand's own implementation, not the tool's surface as a whole.** A different
   shape from 1-3, since a CLI's own `deno install -A -g jsr:@pkg/cli` materializes its WHOLE
   reachable graph for every user regardless of which subcommand they ever run — confirmed real:
   `@zanix/cli`'s own `src/cli.ts` did `import * as commands from './commands/mod.ts'`, a flat
   barrel eagerly importing all 8 commands' own `main.ts` files, so running `zanix new` (needing
   none of it) still materialized `esbuild`/`vite`/`react`/`preact`/`sharp`/`svgo`/`javascript-
   obfuscator` — every heavy dependency `build`/`space` alone needed. This is plain code-splitting,
   not `lazyFunction`/`lazyClass` — there's no "is this feature configured" runtime check to gate on,
   only "did the user invoke this exact subcommand," which the command framework's own dispatch
   already decides. The fix: keep each command's CLI-surface registration (name, flags, `--help`
   text, argument validation) fully eager — breaking that would regress `--help`/tab-completion for
   the WHOLE tool — but split its actual action body into its own file, resolved via a non-literal
   `await import(...)` from inside the action handler, only once that specific subcommand actually
   fires. Confirmed via a real `deno info --json src/cli.ts` before/after: 1219 modules/20 `npm:`
   packages → 395 modules/0 `npm:` packages, with every command's own `--help` output verified
   unchanged. The same duplicated-literal-specifier mitigation (a `specifiers.ts` constant + a
   literal-sync test for any type-position reference that can't use it) applies here exactly as in
   technique 3 — confirmed real for both `npm:esbuild@0.20.2` (4 occurrences across 3 files,
   including 3 unavoidable type-position literals) and `npm:javascript-obfuscator@^4.0.2` in the
   same package.

Don't default to `lazyFunction`/`lazyClass` out of habit once it's been used successfully once in
the ecosystem — confirmed real case: `@zanix/server`'s own TEMP link to `@zanix/utils`'s
`lazyFunction`/`lazyClass`/`lazyValue` was added anticipating technique 3 for the redis-type fix,
but the fix that actually shipped needed neither (a plain `Promise<unknown>` default satisfies it)
— the entire link, plus its paired `scopes`/`compilerOptions.types` override, turned out to be
unnecessary dead weight, confirmed by grepping the package's own real reachable source for
`lazyFunction`/`lazyClass`/`lazyValue` and finding zero hits, then removed. A dead override isn't
just clutter — it caused a real, separate duplicate-module-identity bug (a stale local-path
override resolving `Logger`/`HttpError` to a second, different instance than every other file in
the graph — see the "Known current gaps" section under `@zanix/datamaster`). Check reachability
with a real grep before keeping (or adding) a technique-3 override, not just before adding one.

## The pattern, for a genuinely conditional VALUE/FUNCTION/CLASS

Use `@zanix/utils`'s own `lazyFunction`/`lazyClass`/`lazyValue` (`@zanix/utils/helpers`,
`src/utils/lazy-import.ts`) — don't hand-roll the `await import(specifier)` boilerplate per call
site; that's exactly what these exist to centralize.

```ts
import { lazyFunction, lazyClass, lazyValue } from '@zanix/utils/helpers'

// Function — resolves and calls the real export only when the wrapper is actually invoked.
export const registerJob = lazyFunction<typeof import('./typings.ts').RegisterJobFn>(
  ASYNCMQ_SPECIFIER,
  'registerJob',
)

// Class — can't lazily `new` something you only have after an `await`, so this returns an async
// FACTORY instead of the class itself.
const createConnector = lazyClass<typeof ZanixRabbitMQConnector>(ASYNCMQ_SPECIFIER, 'ZanixRabbitMQConnector')
const connector = await createConnector(options)

// Plain value/constant — a thunk, not cached by the helper itself (Deno's own module cache
// already dedupes repeated `import()` calls to the same resolved specifier — a real ECMAScript
// spec guarantee, not Deno-specific, so don't add a second caching layer on top).
const getDefaultQueueName = lazyValue<string>(ASYNCMQ_SPECIFIER, 'DEFAULT_QUEUE_NAME')
```

Three requirements, all confirmed necessary via a real repro of each partial combination failing:

1. **`specifier` must be a fully-qualified `jsr:pkg@version` (or `npm:pkg@version`) string, kept
   OUTSIDE the caller's own `deno.jsonc` `imports` map entirely** — not a bare alias. A bare alias
   sitting in `imports`, even completely unreferenced by code, is on its own enough to trigger
   materialization.
2. **The specifier must reach the helper as a NON-literal** (a variable/constant, never
   `import('literal-string')` inline) — Deno's own static dependency-graph analysis (and,
   transitively, Vite/Rolldown's own scan during a real build) only follows a dynamic `import()`
   whose argument it can resolve as a literal at parse time; routing it through a variable is what
   keeps the package out of that graph for a consumer that never invokes the wrapper.
3. **Gate the call, not just the import, behind the real "is this feature actually used" check** —
   `if (!entries.length) return` BEFORE the `await import(specifier)` line, not after. The helper
   makes the import lazy; the surrounding code still has to decide when "lazy" means "never."

**If a test file inside the SAME package that owns the specifier needs to resolve the bare alias
directly** (e.g. an internal test importing the real package by name, not through the lazy
wrapper), add it to `deno.jsonc`'s `scopes` field, scoped to that test directory only — a real,
WHATWG-standard Import Maps mechanism (`scopes` overrides the top-level `imports` map for modules
loaded FROM a matching path prefix), not a Deno-specific workaround. Confirmed: this does not
reintroduce the materialization problem for any consumer outside that scope, since `scopes` only
applies to files whose own path falls under the given prefix.

### File/folder convention — where the specifiers and the wrapped resolvers actually live

This isn't a preference — it's the shape that already emerged independently, the same way, across
every real package that needed technique 3 (`@zanix/app`, `@zanix/core`, `@zanix/admin`,
`@zanix/cli`, `@zanix/space`, all already on it as of 2026-08-25) — retrofit the same shape into
any earlier package whose own layout still drifts from it. Follow it exactly, don't invent a fresh
name/location per package.

- **`src/modules/lazy/specifiers.ts`** — ALWAYS at this exact path, one per package, centralized
  regardless of which module(s) actually consume it (never nested under the specific module that
  happens to need it first, e.g. NOT `modules/runtime/lazy-specifiers.ts`). Holds ONLY the
  non-literal specifier constants (`ASYNCMQ_SPECIFIER`, `DATAMASTER_SPECIFIER`, ...) — nothing
  else. **Golden rule: nothing outside this one file ever declares its own specifier constant for
  a technique-3 dependency** — centralizing them here is what keeps a version bump a 1-line change
  in 1 file, the whole reason this file exists. A shim file (below) imports its specifier from
  here; it never declares its own.
  - **Exception, scoped only to a RELATIVE (`./...`) literal-import specifier, never `npm:`/`jsr:`
    ones**: a relative specifier is legitimately collocated with the local files it targets instead
    of centralized here. Confirmed real (`@zanix/space`, 2026-08-26): `not-found-handler.ts`/
    `loader-error-handler.ts` each select a per-renderer default view via
    `getActiveRenderer() === 'preact' ? await import(...) : await import(...)` — the exact
    literal-dynamic-import bug this skill's own `svgo` case already documents, fixed the same way
    (route through a non-literal constant) — but the four constants
    (`DEFAULT_NOT_FOUND_VIEW_REACT_SPECIFIER`, etc.) live in a new
    `modules/router/default-view-specifiers.ts`, not `modules/lazy/specifiers.ts`. This is
    deliberate, not a shortcut: a dynamic `import()`'s relative
    specifier resolves against the CALLING module's own location, never against wherever the string
    constant itself was declared (ordinary ES module semantics) — so a relative specifier would
    still resolve correctly from `modules/lazy/specifiers.ts` too, but a reader opening that
    package-wide file would see a `./default-not-found-view.tsx`-shaped string with no local
    context for what directory it's even relative to. Collocating it next to the router files it
    targets keeps that coupling visible instead of silently correct-but-confusing. An `npm:`/`jsr:`
    specifier carries no such directory coupling — those always still belong in the centralized
    file per the rule above; this exception applies to relative specifiers only.
- **`src/modules/lazy/<dependency>-shim.ts`** — one per mirrored external package. What makes a file
  a "shim" is the promotion criterion below (2+ real consumers sharing a resolver/type), not its
  internal mechanism — it can hold `lazyFunction`/`lazyClass`/`lazyValue`-wrapped resolvers
  importing their specifier from `specifiers.ts`, OR a hand-rolled `await import(SPECIFIER)` wrapper
  when that fits the dependency's shape better (confirmed real: the one shim that actually exists,
  `@zanix/admin`'s `notifications-shim.ts`, uses raw `await import()`, not the helper trio — don't
  read the helper-trio wording as a requirement). It holds that dependency's `*Like` structural
  types too, only when a real type mirror is genuinely needed (prefer `unknown`/a real narrow-subpath
  type import first, see the `import type` precondition section below; don't manufacture a `*Like`
  type just to have one). Created **only when 2 or more consumer files in the package share it** —
  this is what makes the shim worth its own file instead of adding noise.
- **Single-consumer case: stays inline in that one file, full stop — don't create a shim for it.**
  Confirmed real, not hypothetical: `@zanix/datamaster`'s own `CONNECTOR_MODULE_SPECIFIER` lives
  directly inside `storage/core.ts`, the one file that uses it, and `@zanix/app`'s `AsyncmqExports`
  (the narrow hand-declared interface `register-jobs.ts` needs because `lazyFunction`'s specifier
  argument is a variable, so TypeScript can't infer a shape from it — this skill's own validated
  exception to "don't duplicate a contract") lives inline in that same one file too. The moment a
  SECOND consumer needs either one, promote it to the shared shim file above — not before. An
  ecosystem-wide audit (2026-08-26) of every real `lazyFunction`/`lazyClass`/`lazyValue` call site —
  28 total, all in `@zanix/app` and `@zanix/core`, the only two packages that call these helpers at
  all — found zero cases past the threshold: every single one has 0 consumers outside its own
  declaring file, so all 28 correctly stay inline. `@zanix/admin`'s `notifications-shim.ts` remains
  the only package that has ever actually crossed the threshold (2 real consumers, confirmed by
  import grep, not by its own doc comment's claim).

## The `import type` precondition, in practice

Confirmed via direct reproduction, not theory: `import type { Job } from 'jsr:@zanix/asyncmq@^0.7.0'`
— referencing nothing but a type — materializes `amqplib`, `graphql`, and the full `redis`/`@redis/*`
suite, because `@zanix/asyncmq`'s own `deno.jsonc` declares `@zanix/server` (→ `graphql`) and
`@zanix/database`/`@zanix/datamaster` (→ `mongoose`/`redis`) at its own top level, and resolving
`Job`'s real shape requires loading `asyncmq`'s real source, which reaches those.

Two attempts that do NOT work, both empirically ruled out — don't re-try them:

- **Centralizing the `import type` in one file and re-exporting it, hoping to reduce it to "one
  literal specifier total"** — reduces DUPLICATION (a real, separate, legitimate win when the
  source package IS npm-free), but does nothing for materialization if the source package fails
  the precondition above; the file holding the centralized `import type` still eagerly resolves
  the source package's real graph the moment IT is reachable.
- **An ambient "shadow" module declaration** (`declare module "jsr:pkg@version" { ... }` via
  `compilerOptions.types`, intending it to override how the real specifier resolves elsewhere) —
  Deno's own module resolution is independent of TypeScript's ambient-declaration merging; it
  still resolves and materializes the real package regardless of a shadow declaration for the same
  specifier. `compilerOptions.types` genuinely only provides GLOBAL/ambient augmentation (the
  `@types/node`-style shape) — confirmed it does NOT make a normal module's real exports available
  without an explicit import anywhere they're used.

When the precondition fails, in order of preference:

1. **Best: get the source package a genuinely narrow, lean subpath** (see the two-part fix above)
   so the real type becomes safely importable — this is the actual fix, not a workaround.
2. **Interim, if (1) isn't ready yet: a narrow, LOCAL, hand-declared type covering only the fields
   the consuming code actually reads** — not a full mirror of the real type. Real precedent,
   already validated: `@zanix/app`'s own `AsyncmqExports` interface
   (`src/modules/runtime/register-jobs.ts`) types only the two function SIGNATURES it calls, with
   opaque `never` parameters, deliberately not `typeof import('@zanix/asyncmq')` — confirmed that
   even a whole-module `typeof` type alias, despite being erased from emitted JS, still forces the
   same real-source resolution as a value import. **Mark this interim state explicitly temporary**
   (a doc comment stating what unblocks the real fix, e.g. "once `@zanix/asyncmq` publishes a
   `/jobs` subpath, revert to the real type import here") — don't let "local mirror, accept drift"
   read as the final, accepted design. **Do not add an automated periodic structural-drift test
   comparing the local type against the real one** — confirmed rejected as worse than the problem
   it solves: it reintroduces the exact materialization cost (even off the hot path) for a benefit
   no more reliable than the plain literal-string sync test below, at real added tooling
   complexity.
3. When (2) needs to track a real literal specifier string too (a runtime constant AND a
   type-import's literal, for the same package/version — two occurrences that ARE
   mathematically unavoidable together, since an `import`/`import type` specifier can never
   reference a variable, full stop, even within the same file), add a real test asserting the two
   stay in sync — parse both files' own source text for the literal, compare against the runtime
   constant, fail loud on drift. This is a genuine, cheap safety net for an unavoidable
   duplication, not a substitute for centralizing whatever CAN be centralized first.

## A manifest-only check misses a real, confirmed leak: literal specifiers embedded in source

A dependency can resolve an npm/JSR package WITHOUT that package ever appearing in the
dependency's own `deno.jsonc`/`deno.json` `imports` map at all — a plain literal specifier
written directly into a source file's own `import`/`import type` statement, exactly like this
skill's own pattern uses for a lazy value's specifier, but unintentionally, in a file that's
eagerly reachable. **Confirmed real, not hypothetical, in a real, since-fixed case**:
`@zanix/server` (pre-4.0.0) had `import type { RedisClientType } from 'npm:redis@^5.9.0'` in
`src/typings/program.ts`, with NO corresponding `redis` entry anywhere in `@zanix/server`'s own
`deno.jsonc` — grepping the manifest alone found nothing. That file is reached from nearly every
corner of the package (`typings/targets.ts`, `typings/decorators.ts`, `modules/program/mod.ts`,
...), so the literal specifier's reachability was total, not narrow. Fixed by replacing the field's
type with `unknown` (see `@zanix/server`'s own `CoreCacheTypes['redis']` doc for why a concrete
connector's own method override still keeps the real type where it's actually needed) rather than
importing the real type or hand-mirroring its shape. **The precondition check above must grep the
dependency's actual SOURCE for literal `npm:`/`jsr:` specifiers, not only its manifest** — a clean
`deno.jsonc` is not proof the entry file you're about to resolve is clean.

## `mod.ts`/root-export bloat is the sibling bug, not the same one — fix both when found

A package's root `.` entrypoint re-exporting everything (`testing`, `helpers`, `workers`,
`errors`, `validator`, ... all through one `mod.ts`) is a DIFFERENT axis from the `imports`-map
issue above — it's about which SYMBOLS a bare import drags in, pure-JS, independent of npm at
all — but the same "consumer pays for what it doesn't use" instinct applies, and the two bugs are
usually found together (`@zanix/app/runtime`'s barrel, `@zanix/asyncmq`'s `mod.ts`, and
`@zanix/utils`'s own `mod.ts` all had this shape). When auditing a package for the `imports`-map
issue, check its root export surface too.

**Real precedent for how to ship the fix, from `@zanix/utils` (2026-08-25)**: don't default to a
deprecation/shim period. `@zanix/utils` cut its root `.` entirely (deleted `mod.ts`, dropped `"."`
from `deno.jsonc`'s `exports`) as a real `MAJOR` bump, no grace window — a `@deprecated` JSDoc tag
on a barrel re-export doesn't even work as a real signal (TypeScript resolves it against the
ORIGINAL declaration, not the re-exporting barrel, so an editor never flags the bare-import call
site), and "removal deferred to some future major, no committed date" tends to never actually
happen while leaving the duplication live indefinitely for no protective benefit. Before choosing
between a clean break and a deprecation window, **audit every real consumer first** (every
sibling/ecosystem repo, not just the obvious ones) — `@zanix/utils`'s cut was zero-risk specifically
because that audit came back with zero real consumers of the bare import; a package with real
external/JSR-public adoption is a different call, protected by semver caret ranges either way
(an external consumer only feels a `MAJOR` break when they deliberately bump to it).

## A shared, multi-purpose factory/class can bundle unrelated optional capabilities as tightly as a barrel does — check composition, not just re-exports

A third bug shape, related to but distinct from `mod.ts`/root-export bloat above: that one is about
a BARREL FILE re-exporting multiple independent symbols from one entry point (fixed by splitting
the barrel into narrower `exports` subpaths). This one is about a SINGLE factory function or class
that internally, unconditionally WIRES TOGETHER multiple genuinely-separable optional capabilities,
so that using ANY of them forces resolving the heavy dependency of ALL of them — even a narrow
`exports` subpath split doesn't help if the ONE file it points to still does this internally.

Confirmed real (`@zanix/space`, 2026-08-25): `asset-transform/asset-transformer.ts`'s
`createAssetTransformer` factory built one `AssetTransformer` object exposing `transformImage`
(`sharp`-backed), `transformVideo`/`transformThumbnail`/`transformAudio` (FFMPEG-backed, npm-free)
all together, unconditionally wiring in `optimizeImageAsset` as its `transformImage` implementation
even for a caller that only ever calls `transformVideo`. A consumer wanting ONLY video/audio
transcoding (`mediaPlugin`, npm-free on its own) still materialized `sharp` merely by importing the
shared factory file `media-plugin.ts` reached to build its own transformer instance — a narrower
`exports` subpath pointed at THIS SAME shared file would not have fixed anything, since the file
itself, not just the barrel around it, does the unconditional bundling.

**The fix**: split the shared factory into one file per genuinely-separable capability
(`image-transformer.ts`'s `createImageTransformer`, `media-transformer.ts`'s
`createMediaTransformer`, both npm-profile-confirmed via `deno info --json` in isolation), then
recompose the original combined factory as a thin wrapper over both — preserving its existing public
shape/identity (verified via an existing strict-identity test, e.g. `transformImage ===
optimizeImageAsset`, still passing) for any consumer that genuinely needs everything. Only THEN does
splitting the `exports` map into narrower subpaths (`./vite/assets`, `./vite/media`) actually
isolate the two — the file-level split is the load-bearing fix, the subpath split is what lets a
consumer actually reach just one half.

**The tell**: before concluding a narrow `exports` subpath will fix a leak, check not just "does
this subpath's entry FILE avoid touching the heavy dependency" (the golden rule's own check) but
also "does the SPECIFIC FUNCTION/CLASS this subpath exposes internally compose in something that
function's own caller never asked for" — a file can be reachability-narrow while still returning an
object/class instance that's composition-wide.

## A `.tsx` extension alone injects the JSX pragma import — independent of nodeModulesDir, independent of real JSX content

A fourth sibling bug shape, unrelated to every mechanism above: none of them touch the compiler's
own `jsx`/`jsxImportSource` handling, which turns out to have an identical "consumer pays for what
it doesn't use" failure mode, through a completely different trigger.

Confirmed real (`@zanix/space`, 2026-08-26), via a minimal isolated repro, not assumed: a `.tsx`
file compiled under `{"compilerOptions":{"jsx":"react-jsx","jsxImportSource":"react"}}` carries an
implicit `react/jsx-runtime` dependency — as BOTH a `code` and a `type` edge in `deno info --json`'s
own `dependencies` array, at a synthetic `{line:0,character:0}` span (never a real import
statement's own location) — **purely from the file's `.tsx` extension, regardless of whether the
file contains a single real JSX element.** Two otherwise-identical files, one named `.ts` and one
`.tsx`, both containing nothing but `export function noop<T>(x: T): T { return x }` (zero JSX
syntax, only a generic type parameter): `deno info --json --min-dep-age=0` on the `.ts` one shows no
dependencies at all; the `.tsx` one resolves `npm:/react@.../jsx-runtime`. Deno's `.tsx` handling
injects this pragma import from the extension alone, never from first detecting real JSX usage.

The real, confirmed case this surfaced from: `@zanix/space`'s root `.` entry point's own doc claimed
"importing it never evaluates `react`, `react-dom/server` or `preact`" — true for every mechanism
this skill already covers, false in practice because two files reachable from `.`
(`modules/router/space-page-controller.tsx`, `modules/router/not-found-handler.tsx`) were `.tsx`
purely by naming convention, with ZERO actual JSX syntax in either (confirmed by reading full
source — every `<...>` match in both was a TypeScript generic type parameter like
`SpacePageController<Params>`, never a JSX element). Neither file had any reason to pay the
`react/jsx-runtime` cost; nothing in either ever needed JSX transform at all.

**The fix is technique 1 from the decision list above (narrow the entry file itself, no runtime
mechanism needed) — just reached through a mechanism this skill hadn't documented yet.** Renamed
both files `.tsx` → `.ts` (with every import specifier referencing them updated package-wide, since
Deno requires an exact, literal extension in every specifier) — this alone drops the implicit
`jsx-runtime` edge, no other change required. Confirmed via `deno info --json --min-dep-age=0` on
the package's own root entry point, before/after.

**When auditing an entry point for `react`/`preact`/any other JSX-runtime-backed npm leak, grep
every `.tsx` file in its reachable graph for real JSX syntax before assuming the file needs to stay
`.tsx`** — a generic type parameter (`<T>`) or a doc-comment mentioning an HTML tag both look like a
JSX element to a naive `grep '<[A-Za-z]'`; confirm by reading the file, not by grep count alone. A
`.tsx` file that DOES contain real JSX and needs to render under a specific renderer is a completely
different, legitimate case (e.g. `@zanix/space`'s own `default-not-found-view.tsx`/
`default-error-view.tsx`, genuine React components) — the fix there is keeping the renderer's own
files reachable only through the same literal-specifier caution the rest of this skill already
covers (see the companion literal-dynamic-import finding this same investigation found in
`not-found-handler.ts`/`loader-error-handler.ts`, folded into the "File/folder convention" section
above), never renaming a file that genuinely needs JSX.

## TEMP local-path override + `scopes` footguns — split into their own skill

The mechanics of TEMPORARILY linking an unpublished sibling package via a raw relative-path
`imports` override (`scopes` alias-name collisions, the exact prefix depth a `scopes` key needs,
per-subpath override coverage, and a raw local-path override silently auto-linking unrelated
sibling directories too) are a genuinely separate concern from the lazy-dependency pattern itself —
see `deno-workspace-link-pitfalls` for all four, split out from this skill on 2026-08-25 once that
content grew into its own responsibility.

## Known current gaps — real, not yet closed as of this writing

- **`@zanix/asyncmq`'s two-part fix (narrow `/jobs`-shaped subpath + removing
  `@zanix/datamaster`/`@zanix/database` from its own top-level `imports`) is already built and
  confirmed working (2026-08-25).** `asyncmq/deno.jsonc`'s `exports` now has `./worker`/`./core`/
  `./jobs`/`./dlq` alongside `.`; `deno info --json` on `./jobs`'s own entry (`src/modules/jobs/mod.ts`)
  reaches zero npm packages, and the root `.` entry reaches only `amqplib` (no `graphql`/`redis`).
  Not yet published to JSR — still only diagnosable from asyncmq's own unreleased checkout.
- **`@zanix/app`'s `typings/manifest.ts`** already re-exports the real type
  (`export type { CronJobDefinitionBase, Job, JobProcess, ProcessingQueues } from
  '@zanix/asyncmq/jobs'`), not a hand-rolled local mirror — this closed the same day the asyncmq
  fix above landed. What's left is purely a publish-ordering gap: `app`'s own `deno.jsonc` still
  needs to point `@zanix/asyncmq` at the real published version once `asyncmq` actually publishes
  the `/jobs` subpath (today it resolves through a TEMP local-path link to asyncmq's unreleased
  checkout — see `deno-workspace-link-pitfalls`).
- **`@zanix/datamaster` — investigated 2026-08-25, NOT a real problem, false alarm closed.** A
  dedicated isolated repro confirmed `@zanix/datamaster/cache` never materializes `mongoose`/
  `@aws-sdk/client-s3` despite them sitting in the package's shared top-level `imports` — `/cache`'s
  own entry file's real graph never reaches them. `/database` and `/storage` correctly do pull
  `mongoose`/`@aws-sdk/client-s3` respectively, proving the subpaths are genuinely isolated in both
  directions. No fix needed here — the earlier "unconfirmed latent risk" entry this replaces was
  itself based on the same overclaim corrected above (a shared `imports` map entry alone isn't
  evidence of a leak).
- **`@zanix/server` (4.0.0) — GraphQL split into its own `@zanix/server/graphql` subpath
  (`exports` now has both `.` and `./graphql`); the root `.` barrel never exports
  `ZanixResolver`/`Resolver`/`Query`/`Mutation`/`Request` at all (a clean MAJOR break, no
  compat re-export).** `src/modules/infra/handlers/graphql/mod.ts` is the subpath's own entry
  file; confirmed via `deno info` that importing it alone reaches `npm:graphql` but never
  `redis`, and that the package's own REST/Socket/SSR/interactor/connector/provider base classes,
  imported directly, reach neither. `src/typings/program.ts`'s literal `npm:redis@...` type
  reference is gone — `CoreCacheTypes<K>['redis']` is `Promise<unknown>` instead of importing
  `redis`'s own `RedisClientType` (see that type's own doc for why a concrete connector's own
  `getClient` override still gets the real type, unaffected). `Seeders` was already in its own
  file (`typings/general.ts`), separate from the redis reference, by the time this was
  investigated — that specific co-location claim in an earlier version of this entry didn't
  match the real code. **The residual leak this entry used to describe as still open is now
  closed (2026-08-25)**: `modules/webserver/helpers/handler.ts`'s `getMainHandler` — the shared
  REST/Socket/SSR/GraphQL dispatch function `WebServerManager.create()` uses for its default
  handler — had a top-level, unconditional value import of `handlers/graphql/handler.ts`, which
  reaches `npm:graphql` for real, so any REST/Socket/SSR-only consumer of
  `bootstrapServers`/`webServerManager` still materialized `graphql` just by resolving that one
  shared file. `WebServerManager.create()`'s public signature had to stay fully synchronous, and
  `getGraphqlHandler` genuinely can't be deferred past `create()`-call time (it builds the schema
  from resolver metadata a later-finishing multi-call boot sequence can legitimately wipe via
  `ProgramModule.cleanupInitializationsMetadata('postBoot', finalize)` before any request ever
  arrives — confirmed via a real functional test, `graphql-scope-cleanup.test.ts`, that already
  depends on this exact timing) — so a lazy `await import()` at `getMainHandler`-call time was
  never viable here, unlike the ordinary `lazyFunction`/`lazyClass` pattern this skill otherwise
  recommends. The actual fix: a plain registration slot
  (`modules/infra/handlers/graphql/registry.ts`, `registerGraphqlHandlerFactory`/
  `getGraphqlHandlerFactory`, both npm-`graphql`-free) that `handlers/graphql/handler.ts` populates
  as a top-level side effect — which every `@zanix/server/graphql` decorator
  (`Resolver`/`Query`/`Mutation`/`Request`) already triggers transitively (they all import
  `handler.ts` for `getRootValueBucket`/`RequestContext`), matching this package's own already-
  documented usage shape (`docs/handlers.md`'s GraphQL section always showed importing from
  `@zanix/server/graphql` first). `getMainHandler` reads the slot instead of importing
  `handler.ts` directly, and still resolves/calls the real factory synchronously, at the exact
  same point as before. One narrow, deliberate behavior change: a GraphQL server whose default
  handler wiring never had anything reach `@zanix/server/graphql` throws a clear `InternalError`
  instead of silently building an empty stub schema — confirmed via a real, isolated
  `deno cache`/`node_modules` repro that a REST-only entry point (`bootstrapServers`/
  `webServerManager`/`WebServerManager`, all three) now materializes zero npm packages, while
  `@zanix/server/graphql` alone still correctly materializes `graphql`.
- **`@zanix/datamaster`'s own subpaths — the `@zanix/server` pin gap this entry used to describe is
  now closed via a TEMP local link (2026-08-25), pending `@zanix/server` 4.0.0's real publish.**
  `datamaster`'s own `deno.jsonc` still declares `jsr:@zanix/server@^3.0.0` as the eventual real
  target, but overrides the bare `@zanix/server` alias to `../server/mod.ts` (that package's own
  unreleased 4.0.0 checkout) in the meantime — every subpath (`.`, `/cache`, `/database`,
  `/observability`, `/core`, `/sqlite`, `/triggers-api`, `/dlq`, `/dlq-api`, `/storage`, `/files`)
  now uses `@zanix/server`'s bare root, never `@zanix/server/graphql`, so the whole package is
  linked, not a narrow subpath. Confirmed via a real `deno info --json` reachability check (self-
  checking a package's own repo directly is NOT a valid way to test this — see the next bullet) that
  `redis`/`@redis/*` no longer reach `/database`/`/dlq`/`/dlq-api`/`/sqlite`/`/observability`/
  `/triggers-api`/`/storage`/`/files` (only `.`/`/cache`/`/core` legitimately need `redis`), and
  `graphql` no longer reaches ANY subpath, including `.` — a real regression from before this fix,
  where every one of these subpaths pulled both, purely as a side effect of `@zanix/server`'s then-
  pinned `^3.0.0` (resolving `3.3.0`). **Fix needed once `@zanix/server` 4.0.0 actually publishes**:
  drop the local link, bump the real pin to `^4.0.0` — a real, separate, required step, not automatic
  just because `server` publishes.
- **Self-checking a package's own repo directly (`deno check`/`deno test` run from inside it) is NOT
  a valid way to measure per-subpath npm materialization — it installs EVERY npm alias the package's
  OWN top-level `imports` declares, for every subpath, regardless of reachability** (confirmed via
  direct reproduction on `@zanix/datamaster`: checking `/cache` alone, in isolation, still
  materialized `mongoose`/`@aws-sdk/client-s3`/`graphql` into `node_modules/.deno`, contradicting
  this same doc's own earlier, correct finding that `/cache` doesn't reach them). This is the
  "unused bare alias in the CURRENT project's own `imports`" trigger from the golden rule above,
  applying to the package's OWN manifest whenever it is itself the thing being checked/tested — not
  a sign of a real leak. **Use `deno info --json <entry>`'s actual dependency graph (reachable
  `npm:` specifiers) instead** — that reflects genuine reachability regardless of self-check's
  install side effect, and is what actually caught the `@zanix/server`-pin leak above and confirmed
  its fix; a real external-consumer repro (a separate project where the package under test is a
  DEPENDENCY, not the current project) is the only other valid alternative, and is markedly more
  expensive to set up correctly through an unpublished local link (see the next bullet).
- **A raw relative-path `imports` override that replaces a whole package (not a narrow subpath) can
  silently duplicate a shared class used for prototype-patching/`instanceof` — confirmed real via a
  failing test, not theory.** Linking `@zanix/server` to its own unpublished checkout (above) broke
  10 real tests in `@zanix/datamaster`'s own suite (`Logger.prototype.error` spies silently missing
  every call) because `@zanix/server`'s OWN `deno.jsonc` carried an `@zanix/helpers` override
  pointing at `@zanix/utils`'s own unpublished checkout too, and files reached only through THAT
  override (`utils/cron.ts`, `helpers/masking/hard.ts`) resolved `Logger`/`HttpError` to that
  checkout's own internal files — a second, separate module instance from the real
  `jsr:@zanix/utils@^3.0.0/logger`/`/errors` one `@zanix/datamaster`'s own `@zanix/logger`/
  `@zanix/errors` aliases (and everything else in its graph) resolve to. **The fix is NOT to chase
  every individual internal file with a `deno.jsonc` `scopes` pin** (`@zanix/server`'s own
  `"../utils/src/"` scope only pins the two entry-point specifiers `modules/logger/mod.ts`/
  `modules/errors/main.ts` — real internal files reached by a DIFFERENT bare path, like
  `modules/logger/main.ts`/`base.ts`, silently bypass it, and there is no general "whole directory"
  import-maps mechanism to redirect an internal file tree onto a package's own single public
  subpath). **Fix at the actual override that shouldn't exist**: audit whether the override still
  needs to point locally at all — `@zanix/datamaster`'s case confirmed `@zanix/server`'s own real
  source no longer uses anything from the unpublished `@zanix/helpers` checkout unavailable in the
  real published `jsr:@zanix/utils@^3.0.0/helpers` (a stale leftover from an earlier draft of that
  package's own fix), so a single `scopes` entry in the CONSUMING package
  (`"../server/src/": {"@zanix/helpers": "jsr:@zanix/utils@^3.0.0/helpers"}`) — pinning the override
  itself back to the real published subpath, scoped to files under the linked package's own source
  tree — closed the whole class of duplication at once, without touching the linked package's own
  (out-of-scope, sibling-repo) `deno.jsonc`.
- **A `compilerOptions.types` override pointing at another package's own raw ambient `.d.ts` file
  has the identical duplication risk, and it's easy to miss because the import is `import type`,
  inside a file never reached by the normal module graph — the compiler processes it anyway.**
  `@zanix/app`/`@zanix/server`/`@zanix/datamaster` all point `compilerOptions.types` at
  `../utils/src/typings/index.d.ts` directly (to declare the `Znx` ambient global — real JSR
  packages can't inject a global declaration into consumers automatically) — that file does
  `import type { Logger } from 'modules/logger/main.ts'`, resolved against `@zanix/utils`'s OWN
  unpublished checkout, independently reintroducing the exact `Logger`/`HttpError` duplication
  above regardless of any `scopes` fix elsewhere. **The correct fix, matching `@zanix/utils`'s own
  documented intended usage** (see `ZanixGlobal`'s own doc, `typings/zanix.ts`): a small LOCAL
  `declare global` file in the CONSUMING package, importing `ZanixGlobal` through the real published
  `jsr:@zanix/utils@^3.0.0/types` (or that package's own equivalent `@zanix/types` alias) instead of
  a raw path into the source tree — zero reach into any unpublished checkout, so no duplication risk
  regardless of what else in the graph still points locally. `@zanix/datamaster`'s own
  `src/typings/zanix-global.d.ts` is the real fixed precedent; `@zanix/app`'s and `@zanix/server`'s
  own identical raw-path overrides carry this same latent risk, unaudited as of this writing.
- **`@zanix/datamaster` — new narrow `./dlq` subpath added (2026-08-25)**, mirroring `/cache`'s
  isolation from `/database`/`/storage`: root `mod.ts` was bundling `DlqProvider`/
  `ZanixCoreDlqProvider`/`DlqAdminService`/etc. into the same barrel as the cache module, so
  importing `DlqProvider` alone forced resolution of the full `redis`/`@redis/*` stack as a pure
  side effect of barrel co-location — not because DLQ's own code touches cache. `./dlq`'s own
  entry file confirmed clean of `modules/cache/` (code and type edges both), with a permanent
  `deno info --json`-based regression guard added in `datamaster`'s own test suite. Narrowing the
  LOCAL footprint and closing the cross-package `@zanix/server` leak were two separate,
  independently-required fixes — both now done (see the `@zanix/server` pin bullet above).
- **`@zanix/asyncmq`'s `/dlq` subpath should migrate** from its current literal
  `jsr:@zanix/datamaster@^1.0.0` (root) specifier to `@zanix/datamaster`'s new `./dlq` subpath once
  a `datamaster` release with it ships — not yet applied (unpublished, sibling-repo coordination).

## Checklist before declaring a dependency "safely lazy"

- [ ] When reading `deno info --json`'s own output, checked per-entry REACHABILITY (the `modules`
      array's own graph edges — `dependencies[].code.specifier`/`dependencies[].type.specifier` —
      walked from the specific entry file being probed), not a package-wide summary field?
      **Confirmed real gotcha (`@zanix/cli`, 2026-08-25)**: probing each of 8 CLI commands' own
      `main.ts` in isolation first appeared to show all 8 pulling the identical, full 20-`npm:`-
      package set — a measurement artifact, not a real finding, because every command file
      transitively re-imports every OTHER command through a shared barrel, and a naive read of the
      JSON's own aggregate npm-package listing reflects the whole resolved CONFIG's npm closure, not
      what's actually reachable from the one entry file being checked. Tracing the real graph edges
      (not the aggregate field) found only 2 of the 8 commands were genuinely heavy. Don't trust a
      `deno info --json` result's own summary-shaped field for a per-entry-point claim without
      confirming it's built by walking that specific entry's own edges.
- [ ] Traced which entry file the specifier actually resolves to (the bare `.`, or the specific
      subpath being imported), and checked THAT file's own real import graph — not just whether
      the package's shared `imports` map has an entry for something heavy somewhere?
- [ ] If writing `import type`, checked the resolved entry file's own source (grep it directly,
      not just `deno.jsonc`) for a literal `npm:`/`jsr:` specifier the manifest wouldn't show?
- [ ] `lazyFunction`/`lazyClass`/`lazyValue` used instead of hand-rolled `await import()`
      boilerplate, unless there's a specific documented reason not to (e.g. `AsyncmqExports`'s
      narrow-interface pattern for a case a generic helper doesn't fit)?
- [ ] Specifier is a non-literal (variable) at the call site, and absent from the caller's own
      `imports` map — verified via a real `deno check`/`node_modules` repro, not assumed from the
      code shape alone?
- [ ] The actual capability gated (`if (!x.length) return`) BEFORE the lazy import runs, not just
      the import itself made lazy?
- [ ] If a narrow-subpath fix is the real target, checked whether the package's own top-level
      `imports` ALSO needs the same lazy treatment internally — not just the new subpath's
      `exports` entry?
- [ ] Traced one layer further up/down the dependency chain for the same shape, not stopped at the
      first package fixed?
- [ ] If a `react`/`preact`/other JSX-runtime-backed npm package shows up in a reachable entry
      point's graph, checked every `.tsx` file in that graph for REAL JSX syntax (by reading it, not
      grep count alone) before assuming it needs to stay `.tsx` — a `.tsx` extension alone injects
      the JSX pragma import under a global `jsxImportSource`, independent of whether the file
      actually contains JSX?

## Out of scope — do not do these

- Recommending a change to `nodeModulesDir` as a fix for this class of bug — confirmed
  workspace-root-only and provably invisible to consumers; only relevant as unrelated internal
  hygiene for a package with zero npm dependencies of its own (guards against a future accidental
  npm addition going unnoticed), never as the actual fix here.
- Adding an automated periodic structural-drift test comparing a local interim type against the
  real upstream one — confirmed rejected; a literal-string sync test (step 3 above) is the
  accepted safety net for the cases that genuinely need one.
- Deciding remediation order/timeline across multiple packages found to need this pattern — report
  what's confirmed vs. still-latent-and-unconfirmed (see "Known current gaps"); sequencing which
  package gets fixed first is a human call.

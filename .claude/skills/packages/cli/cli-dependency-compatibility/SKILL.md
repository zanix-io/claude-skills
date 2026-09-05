---
name: cli-dependency-compatibility
description: How @zanix/cli keeps generated code compiling against real, currently-published @zanix/* package versions — the pinned-version source of truth, the scheduled Drift Watch CI check, and the opt-in --verify flag — plus a separate concern, cli's own deno.jsonc native-runtime-module declarations that zanix space dev's SSR pipeline needs to resolve a @zanix/space NATIVE_RUNTIME_MODULES specifier at all. Use when adding a new @zanix/* import to a generator/template, investigating a --verify/Drift Watch failure, or adding a package to @zanix/space's own NATIVE_RUNTIME_MODULES.
---

This skill covers a problem specific to a code generator: `cli`'s generators
are tested for *deterministic output* (the generated source matches an
expected snapshot — see `cli-generator-testing`), which by itself says nothing
about whether that output still *compiles* against the real, currently-published
version of whatever it imports. File:line references point at
`~/Documents/Development/ZanixLibraries/cli` — read the real code there before
assuming this summary is still accurate.

Packages a generated project can end up depending on: `@zanix/server`
(and its `/graphql` subpath, `--type graphql` handlers only),
`@zanix/validator`/`@zanix/types` (aliases, see below), `@zanix/datamaster`,
`@zanix/asyncmq` (and its `/jobs` subpath), `@zanix/core`, `@zanix/app` (and
its `/runtime` subpath), `@zanix/space`, `@zanix/space-ui` (`--icons`/
`--theme astronaut` only), `@zanix/utils/logger`/`@zanix/utils/linter`
(subpath aliases, same treatment as `@zanix/validator`/`@zanix/types`).

## Golden rule (token savings)

- **`deno check` scoped to what changed, most of the time.** Run it against
  the whole generated project only for the final Validation step (or when
  actually investigating a real drift failure) — while iterating on one
  generator's templates, checking just the files that generator writes is
  enough and far cheaper.
- **Checking whether a package needs the alias treatment is a one-line lookup
  in `ZANIX_DEPENDENCY_VERSIONS`, not a fresh investigation** — the answer
  for `@zanix/validator`/`@zanix/types` is already established; only check a
  *new* package name you haven't seen in that table before.
- Don't re-run Drift Watch's own script locally to "double check" a
  `--verify` result — they run the identical check; a `--verify` failure
  already tells you everything Drift Watch would.

Three layers mitigate this — all independent, all worth checking when
something looks version-related.

## Layer 1 — generate against known-compatible dependency versions

`src/utils/config/dependencies.ts` is the single source of truth for every
`@zanix/*` version `cli` ever writes:

- `ZANIX_DEPENDENCY_VERSIONS: Record<string, string>` — one entry per package,
  the exact import specifier written into a generated project's `imports`.
  Bumping a compatible version is a one-line edit here — nothing else in `cli`
  hardcodes a version of its own.
- `PROJECT_TYPE_DEPENDENCIES: Record<ZanixProjects, string[]>` — which of those
  packages each project type's scaffold actually imports, verified against the
  real generator/template output, not assumed: `library` → none; `app` →
  `@zanix/app`/`@zanix/utils/logger`; `space` →
  `@zanix/space`/`@zanix/app/runtime`; `server` →
  `@zanix/server`/`@zanix/datamaster`/`@zanix/asyncmq`/`@zanix/asyncmq/jobs`/
  `@zanix/validator`/`@zanix/core`; `space-server` →
  `@zanix/space`/`@zanix/server`/`@zanix/datamaster`/`@zanix/asyncmq`/
  `@zanix/asyncmq/jobs`/`@zanix/validator`/`@zanix/core` — **not** the union of
  `space`'s and `server`'s own arrays: it excludes `@zanix/app/runtime`,
  because `space-server`'s root `mod.ts` calls `Zanix.start()` directly
  instead of going through `bootstrapRemoteApp`. `server`/`space-server`
  need `@zanix/core` because their scaffold includes a real, runnable root
  `mod.ts` that calls `Zanix.start()` — not because any example artifact file
  imports it directly. `@zanix/asyncmq/jobs` is separate from the bare
  `@zanix/asyncmq` entry because `server`'s/`space-server`'s own tree seeds an
  `example-job.defs.ts` (`planJob`) that imports `registerCronJob` from that
  subpath specifically. `baseZnxConfig` writes exactly this list into a
  freshly generated `deno.json` — no more, no less, no implicit versions.
- `ensureZanixDependency(root, pkg)` — the `zanix generate` counterpart, for
  adding one artifact to an already-scaffolded project. Same never-clobber
  guarantee as `ensureConstant`: adds `pkg`'s import only if `deno.json`
  doesn't already declare it, never overrides a version the project owner
  pinned by hand. Every `generate/*/command.ts` that needs a `@zanix/*`
  package calls this after writing its files.

**Alias packages — `@zanix/validator` and `@zanix/types` are not published
packages of their own.** Both are aliases into `@zanix/utils`'s own
`/validator`/`/types` subpaths (`ZANIX_DEPENDENCY_VERSIONS`'s entry for
`@zanix/validator` points at `jsr:@zanix/utils@2.*/validator`, matching the
convention every sibling Zanix repo uses for the same import). Before adding a
new `@zanix/*` name to this table, check whether it's a real package or another
alias into an existing one — treating an alias as a standalone package would
write an import specifier that doesn't resolve.

## Layer 2 — scheduled Drift Watch CI

`.github/workflows/drift-watch.yml` runs on a weekly schedule, on every push to
`master`, and on manual dispatch. Delegates to `scripts/drift-watch.ts`, which:

1. Runs `zanix new` for every project type (`library`/`app`/`space`/`server`/
   `spacecraft`) into a temp dir, plus a curated `zanix generate` variant
   matrix against a fresh `server` project (every `handler --type`, `connector
   --slot` shape, `job` with/without `--cron`, every `middleware --kind`, a
   multi-type `rto --field` spread, `dlqprocessor`, etc.) and a fresh
   `space --icons` project (`comet`/`component`/`page`/`layout`/`interactor`).
   `--icons` matters here beyond just being one more flag: `@zanix/space-ui`
   is a real `ZANIX_DEPENDENCY_VERSIONS` entry consumed ONLY via the
   `--icons` scaffold path (`ensureSpaceUiDependency`,
   `commands/new/lib/tree/projects/space-icons.ts`) — a plain
   `zanix new space` with no flags never declares or imports it, so this is
   the only place Drift Watch ever exercises that package's real published
   API at all. `--type`/`--kind`'s variant lists are imported directly from
   `handler/command.ts`'s own `HANDLER_TYPES` and `middleware/command.ts`'s
   own `MIDDLEWARE_TYPES` (real closed enums) rather than hand-duplicated, so
   those axes can't drift from the generators they watch; `--slot`/`--field`
   accept open-ended strings with no closed set to derive from, so those
   variants are curated by hand to hit every distinct code path instead.
   **The lesson, generalized**: a new `@zanix/*` dependency that's reachable
   only through an opt-in flag (not a project type's base scaffold) needs its
   own explicit Drift Watch variant — it's invisible to every other variant
   in the matrix otherwise, confirmed real (this is exactly the gap
   `@zanix/space-ui`/`--icons` had until fixed).
2. Rewrites each generated project's `deno.json` to the **real latest
   published JSR version** of every package in `ZANIX_DEPENDENCY_VERSIONS` —
   resolved live via `https://jsr.io/<pkg>/meta.json` — instead of `cli`'s own
   pinned range, so this actually tests "what's live on JSR right now." A
   package that can't be resolved (not published yet) is left on `cli`'s
   pinned range; nothing this script can fix either way.
3. Runs `deno check` against every file in each generated project and reports
   pass/fail per project/variant group.

**Informational, not blocking** — never a required check, never gates
`publish.yml`. A red run means an upstream Zanix package changed in a way that
broke generation (or hasn't published yet), not that `cli` itself regressed.
Notification is the workflow run itself (red in the Actions tab) — no
auto-filed issue, no external webhook, by deliberate choice (the
lowest-maintenance option for a repo without an established issue-triage or
chat-notification convention to hook into).

## Layer 3 — `--verify`: on-demand, scoped, opt-in

`--verify` (`src/utils/verify.ts`'s `verifyGeneratedProject`) is available on
`zanix new` and every `zanix generate <artifact>`. After generation, it runs
`deno check` against every file in the (whole) project and logs a warning —
**never changes the command's own exit code** — if it doesn't compile against
whatever dependency versions are actually resolvable right now. Same
underlying check as Drift Watch, but on-demand and scoped to one project
instead of every variant.

**Deliberately opt-in, not default-on**: `zanix new`/`zanix generate` are 100%
local and instant by default (no network dependency), and making every
invocation also run a real `deno check` against JSR would be a genuine UX
regression — slower by default, and a new failure mode in offline/CI-sandboxed
environments where generation currently works fine.

**A `--verify` failure never means generation itself is wrong** — the
generated code is still correct against `cli`'s own known API shape. It means
an upstream Zanix package changed in a way that broke it, or hasn't published a
version yet.

**Sharp edge if this gets touched again**: `deno check`'s own config-file
discovery resolves from the *calling process's* cwd, not from the paths of the
files being checked — omitting an explicit `cwd: root` on the `Deno.Command`
silently checks the generated project against `cli`'s own `deno.jsonc` instead
of the generated project's, giving a false pass/fail entirely.

## A separate concern: cli's own `deno.jsonc` — native-runtime-module declarations (`zanix space dev`)

Everything above is about what a **generated project** imports. This section
is about a completely different thing: what `cli`'s **own running process**
needs to resolve, for itself, while `zanix space dev` is up.

`@zanix/space`'s own `nativeRuntimeModulesPlugin`
(`native-runtime-modules.ts`, its internal `NATIVE_RUNTIME_MODULES` array)
routes a fixed list of bare specifiers — `@zanix/space`, `@zanix/server`,
`@zanix/auth`, `@zanix/datamaster`, `@zanix/asyncmq`, `@zanix/notifications`,
plus `react`/`react-dom`/`preact`/`preact/hooks` — through
`RealImportEvaluator.runExternalModule` (`ssr-module-evaluator.ts`), which does
a **plain native `import(specifier)`**, deliberately never resolved against
the scaffolded project's own `deno.json` — the whole point is that this must
share real reference identity with whatever `cli`'s own process already
loaded (see that file's own doc for the module-identity bug this closes: two
separately-evaluated copies of the same package silently split shared
registries/DI containers).

That `import()` runs inside the process `zanix space dev` itself started, so
it resolves against **`cli`'s own governing `deno.jsonc`** (see
`import-project-module.ts`'s own module doc for why one `deno run <entry>`
invocation shares one governing resolver, rooted at the entry's own config —
never per-file). If a package on `@zanix/space`'s list has no entry in `cli`'s
own `deno.jsonc` `imports`, that native `import()` fails outright with `Import
"<pkg>" not a dependency and not in import map` — a real, reported crash
(confirmed against `@zanix/notifications`/`@zanix/datamaster` on published
`@zanix/space@1.1.0`/`1.2.0`, fixed by adding both to `deno.jsonc`, mirroring
the entry `@zanix/auth` already had).

**This is not the same fix as `ZANIX_DEPENDENCY_VERSIONS`/
`PROJECT_TYPE_DEPENDENCIES` above.** Those govern a *generated* project's own
`deno.json` — a separate file, a separate resolver, a separate process
entirely. `cli` itself never imports most of these packages directly; the
`deno.jsonc` entry exists purely so this one native `import()` has something
to resolve against, the same reason `preact`/`react`/`@zanix/auth` already
carry one (each entry documents this at its own site — read `@zanix/auth`'s
for the fullest version of this reasoning).

**Checklist — whenever `@zanix/space`'s own `NATIVE_RUNTIME_MODULES` gains a
new package** (or when investigating a `"not a dependency and not in import
map"` crash reported against `zanix space dev`):

- [ ] Is the new package declared in `cli`'s own `deno.jsonc` `imports`, with
      a comment explaining why (the native-runtime-module identity reason,
      not a generator-template reason)?
- [ ] Does `src/@tests/integration/commands/space/dev/
      native-runtime-module-imports.test.ts` list it? That test does a real
      dynamic `import()` of every `@zanix/*` package on this list against
      `cli`'s own config — the regression guard for this exact gap.
      `@zanix/space` now exports `NATIVE_RUNTIME_MODULES` itself (`./dev`,
      added specifically to close this hand-sync gap) — once `cli`'s own
      `@zanix/space` range picks up the version that first ships it, switch
      this test to import the REAL list (filtered to `@zanix/*` entries)
      instead of a hardcoded copy (see that test file's own TODO). Until
      then, the hardcoded list must still be kept in sync BY HAND.
- [ ] Was `deno.lock` regenerated (`deno install`) after editing `imports`?

## Checklist before adding a new `@zanix/*` import to any generator/template

- [ ] Is the package registered in `ZANIX_DEPENDENCY_VERSIONS`? Is it a real
      published package, or does it need the alias treatment
      (`@zanix/validator`/`@zanix/types`-style) instead?
- [ ] Is `PROJECT_TYPE_DEPENDENCIES` updated for every project type whose
      scaffold now actually imports it, verified against the real
      generator/template output — not assumed?
- [ ] Does the generator's `command.ts` call `ensureZanixDependency` after
      writing its files, so `zanix generate` (not just `zanix new`) declares
      the dependency too?
- [ ] Would this new import need a new Drift Watch variant to exercise its own
      code path — and if the axis has a closed set (like `handler --type`), is
      it imported from the generator's own source rather than hand-duplicated?

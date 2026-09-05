---
name: cli-global-install-mechanics
description: How `deno install -g jsr:@zanix/cli` actually resolves dependencies at runtime — the synthetic shim config it generates never consults @zanix/cli's own published deno.jsonc unless --config is passed explicitly (and that --config must carry every field the shim's process-wide behavior depends on, nodeModulesDir included, not just imports/minimumDependencyAge), and the shim's own lockfile only covers whatever mod.ts statically imports, leaving a command whose real body lives behind a dynamic import() to resolve fresh at runtime regardless of @zanix/cli's own minimumDependencyAge. Use when a real global install of @zanix/cli rejects a freshly-published/npm-cached dependency, throws a Deno-native "Cannot find module...verify main entry" CJS error deep inside a served project's own npm graph, or when reviewing/extending src/installation/setup.ts.
---

This skill is about how `deno install -g` treats `@zanix/cli` specifically —
not about the resolver logic that rewrites a SERVED project's own bare
specifiers once `zanix space dev`/`build` is running (see
`deno-workspace-link-pitfalls` for the general `@deno/loader` `Workspace`
gotchas that resolver also hits). File:line references point at
`~/Documents/Development/ZanixLibraries/cli` — read the real code there
before assuming this summary is still accurate; it will drift as the CLI
evolves.

## Golden rule (token savings)

- Before assuming a global-install dependency-resolution failure is a bug in
  `@zanix/cli`'s own resolver logic, check whether it's actually one of the
  three mechanics below — a missing `--config`, a `--config` that's missing
  a specific field (`nodeModulesDir` confirmed, possibly not the only one),
  or a dynamically-imported command's own dependency outside the shim's
  lockfile. All three produce a surface symptom ("not a dependency and not
  in import map", a freshness-window rejection, or a Deno-native CJS
  "Cannot find module" error) that reads like a genuine resolver bug, and
  are cheaper to rule out first.
- `src/installation/setup.ts` is the fix for both, already shipped — check
  it's actually being used (`deno run -A jsr:@zanix/cli@[version]/setup
  [version]`) before re-diagnosing either mechanic from scratch.

## `deno install -g`'s synthetic shim config never reads the package's own `deno.jsonc`

`deno install -g jsr:@zanix/cli` generates a shim at
`~/.deno/bin/.zanix/deno.json` — a config file `deno install` itself writes
for the shim to run under, **not** a copy of `@zanix/cli`'s own published
`deno.jsonc`. A plain install with no `--config` flag leaves this file
holding little more than `{"workspace": []}` (confirmed via a real install)
— none of `@zanix/cli`'s own `imports` map (`minimumDependencyAge`, every
`jsr:`/`npm:` entry the package's own native `import()` calls need) makes it
into the shim at all.

The practical effect: any native `import()` inside `@zanix/cli`'s own code
against a bare specifier the package declares — `@zanix/server`'s own
static import inside `commands/space/dev/action.ts`, `react/jsx-dev-runtime`
natively imported by `@zanix/space`'s SSR pipeline — has nothing to resolve
against once running from a plain global install, and fails outright.

**The fix**: pass `--config` to `deno install` itself, pointing at
`@zanix/cli`'s own real `imports` map (filtered to scheme-based entries
only — the package's own internal local aliases like `typings/`, `shared/`
would resolve against wherever the temp config file sits, not the package's
real source tree). Its content MERGES into the synthetic shim file rather
than replacing it. `src/installation/setup.ts` does exactly this: fetches
`https://jsr.io/@zanix/cli/${VERSION}/deno.jsonc`, filters `imports` to
`/^(jsr:|npm:|https?:)/` values, writes a temp config carrying that plus
`minimumDependencyAge`, and passes it as `configArgs` to `deno install`
(`src/installation/setup.ts:119-186`) — best-effort: a fetch/parse failure
warns and proceeds without it rather than failing the whole install.

## The filtered `--config` must carry EVERY field the shim's process-wide behavior depends on — `nodeModulesDir` included, not just `imports`/`minimumDependencyAge`

A real, confirmed instance of the SAME "the shim's config is a separate file the filter step must remember to populate" gap the section above describes — this one cost a full production build failure to find, so it earns its own section rather than being folded into a bullet point.

`@zanix/cli`'s own `deno.jsonc` sets `"nodeModulesDir": "auto"` (`deno.jsonc:5`) — but `setup.ts`'s config-filter block, before this was fixed, only ever carried `minimumDependencyAge` and `imports` into the shim's `--config` file, silently dropping `nodeModulesDir`. **This matters because `nodeModulesDir` is a process-wide Deno setting tied to the RUNNING PROCESS's own governing config, never the config of whatever project that process happens to be serving** — `zanix space build`/`dev` resolves EVERY npm dependency it touches (including one deep inside a SERVED project's own dependency graph, several `jsr:`/`npm:` hops away from anything `@zanix/cli` itself declares) against the shim's own `nodeModulesDir` setting, not the served project's, even when that project sets `nodeModulesDir: "auto"` itself and has a real local `node_modules` tree on disk.

**Confirmed live, real production failure** (`aeratech-console`, a `@zanix/space` consumer built entirely on `2.0.9-alpha.2`, `--config` filter still missing `nodeModulesDir`): `zanix space build` threw `Cannot find module '.../react-remove-scroll-bar/2.3.8/dist/es5/constants.js'. Please verify that the package.json has a valid "main" entry` — a genuine Deno-native Node-compat CJS error (the exact string is baked into the `deno` binary itself, confirmed via `strings`), three `npm:` hops deep inside `@zanix/space-ui`'s `Modal` → `@radix-ui/react-dialog` → `react-remove-scroll` → `react-remove-scroll-bar`'s own compiled code — entirely outside anything `import-project-module.ts`'s own rewriter ever touches (that module only rewrites the SERVED PROJECT's own source tree; once a scheme-carrying `jsr:`/`npm:` specifier resolves, native execution takes over completely, with zero further rewriting). `react-remove-scroll-bar` ships a legacy "private stub subpath" package — a `constants/` folder holding only a `package.json` whose `"main"` escapes its own directory via `../dist/es5/constants.js` — a real, valid, pre-`exports`-field npm convention. Verified via direct, isolated `createRequire` repros: this pattern resolves CORRECTLY when invoked from a file physically inside the project's own local `node_modules/.deno/<pkg>@<version>/node_modules/...` vendored tree, and FAILS when resolution falls back to Deno's flat GLOBAL npm cache instead (the exact path the real error cited) — which is precisely what happens once the shim's own config lacks `nodeModulesDir: "auto"`.

**Verified as the actual, sole cause, both directions**: manually adding `"nodeModulesDir": "auto"` to a live shim's `~/.deno/bin/.zanix/deno.json` turned an identically-reproducing failing build green instantly (full client build, every route rendered, zero errors); removing it again reproduced the exact same failure. No other change was involved either time.

**The fix**: `setup.ts`'s config-filter block now also copies `nodeModulesDir` from the published config into the filtered `--config` file (`src/installation/setup.ts:119-160`ish — check the current line range, this section moves as the file changes). **The general lesson, not just this one field**: any setting `@zanix/cli`'s own `deno.jsonc` declares that governs how the RUNNING SHIM PROCESS resolves/behaves (as opposed to something scoped to `@zanix/cli`'s own source tree specifically) is a candidate for this same silent-drop gap — `nodeModulesDir` was the first one found the hard way, not necessarily the last. When adding a new top-level field to `@zanix/cli`'s own `deno.jsonc` that affects process-wide Deno behavior, check whether `setup.ts`'s filter block needs to propagate it too, rather than assuming `imports`/`minimumDependencyAge` are the only two that matter.

## The shim's lockfile only covers the STATIC import graph — a dynamically-imported command's own deps resolve fresh at runtime

`deno install -g` also generates a lockfile for the shim
(`~/.deno/bin/.zanix/deno.lock`), but only for what `mod.ts`'s own STATIC
import graph reaches. `@zanix/cli`'s own command dispatch is deliberately
lazy: `commands/mod.ts` eagerly imports every command's own registration
module just to expose its CLI surface (name/options/description, needed for
`--help`), but each command's real body lives behind a genuinely DYNAMIC
`import()` — a variable specifier, never an inline literal, specifically so
Deno's own static dependency-graph analysis can't trace it and pull every
OTHER command's own heavy dependencies (Vite, React, `sharp`,
vanilla-extract, ...) into the eager registration graph just from resolving
`--help`. See `commands/space/dev/command.ts:11-36`'s own doc comment for
the full mechanism and why even a `typeof import(...)` type alias would
defeat the purpose (still forces the same real-source resolution as a value
import, despite being type-only and erased from emitted JS).

The consequence: a command whose body is dynamically imported (`zanix space
dev`/`build`'s own `action.ts`) resolves ITS OWN dependencies FRESH at
runtime, the moment a user actually runs that command — not at install
time, and not covered by the shim's own generated lockfile. That fresh
resolution is subject to Deno's default 24h dependency-freshness window
regardless of what `@zanix/cli`'s own `deno.jsonc` sets
`minimumDependencyAge` to (`--config` from the section above fixes the
IMPORT MAP gap, but a lockfile entry is a separate thing — the shim's own
lockfile still won't have pinned a dependency the static graph never
reached). Installing right after a fresh release, or right after a
dependency the command needs gets updated, throws a same-day-freshness
rejection even though the shim's own config now correctly declares
`minimumDependencyAge: 0`.

**The fix**: `src/installation/setup.ts`'s lockfile-sync step
(`src/installation/setup.ts:199-237`) fetches `@zanix/cli`'s own published
`deno.lock` (generated from a full `deno test` run that DOES reach every
command's dynamically-imported dependencies) and merges its
`specifiers`/`jsr`/`npm` sections into the shim's own lockfile — add-only,
never overwriting an entry the install step already resolved. Best-effort,
same as the config-merge step: a fetch/merge failure warns rather than
failing the install, since the alternative (rejecting a fresh dependency the
next time that specific command runs) is recoverable with a retry, not a
broken install.

## The `deno run` invocation that resolves `setup.ts` itself is a THIRD, separate freshness-window exposure

Distinct from both gaps above — those are about dependencies `setup.ts`
installs; this one is about resolving `setup.ts` itself. `deno run -A
jsr:@zanix/cli@[version]/setup [version]`, with no flag on the OUTER `deno
run`, fails outright for any version published within Deno's own default
24h freshness window — confirmed live: `deno run -A
jsr:@zanix/cli@2.0.9-alpha.2/setup 2.0.9-alpha.2`, run against a version
published under 24h earlier, failed before `setup.ts` ever ran a single
line, with `Could not find version of '@zanix/cli' that matches specified
version constraint... blocked by the minimum dependency age policy`. The
`--minimum-dependency-age 0` `setup.ts` passes to its own INTERNAL `deno
install` call (the section above) has no effect here — that flag only
applies to the process `setup.ts` itself spawns, not to the outer `deno run`
that fetches and runs `setup.ts` in the first place.

**The fix**: add `--minimum-dependency-age 0` to the outer command, but only
for a version genuinely published within the last 24 hours — this is a
CONDITIONAL fix, not baked into the canonical documented command
permanently. Hardcoding it into the one command every user runs, regardless
of how old the version they're installing actually is, would silently
disable the freshness safety net for every install forever, not just the
fresh-release edge case (a real security-posture regression this skill
almost shipped, caught before being committed):

```sh
deno run -A --minimum-dependency-age 0 jsr:@zanix/cli@[version]/setup [version]
```

Both `README.md`'s Installation section and `setup.ts`'s own module doc
(`src/installation/setup.ts:8-14`) document the flag this same way — as an
add-it-if-you-hit-the-error note beneath the base command, never folded
into the base command itself. Check both still show it that way before
trusting either as current.

## Checklist before diagnosing a global-install dependency-resolution failure as a resolver bug

- [ ] Was `@zanix/cli` installed via `src/installation/setup.ts` (`deno run
      -A jsr:@zanix/cli@[version]/setup [version]`), or a plain `deno
      install -g jsr:@zanix/cli` with no `--config`? The latter leaves the
      shim's own config nearly empty — check `~/.deno/bin/.zanix/deno.json`
      directly before assuming a resolver bug.
- [ ] Does the shim's own `~/.deno/bin/.zanix/deno.json` actually carry
      every process-wide field `@zanix/cli`'s own `deno.jsonc` sets
      (`nodeModulesDir` confirmed real — check for others), not just
      `imports`/`minimumDependencyAge`? A Deno-native "Cannot find module...
      verify main entry" CJS error, reached several `npm:` hops deep inside
      a SERVED project's own dependency graph, is this gap — not a bug in
      whatever package the error names.
- [ ] If the failure happens before `setup.ts` runs at all (a `deno run`-
      level version-constraint error citing "minimum dependency age," not a
      `zanix-installer` log line), was the version being installed
      genuinely published within the last 24 hours? If so, `--minimum-
      dependency-age 0` on the OUTER `deno run` command is the fix — but
      don't add it speculatively when the version isn't actually that
      fresh, since it disables a real safety net for no reason.
- [ ] Is the failing specifier reached through a command's own dynamically-
      imported `action.ts` (e.g. `zanix space dev`/`build`), rather than
      something `mod.ts`'s eager registration graph already resolved at
      install time? If so, check whether the shim's own lockfile
      (`~/.deno/bin/.zanix/deno.lock`) actually has an entry for it — a gap
      there means the freshness window applies regardless of
      `minimumDependencyAge`.
- [ ] Does the failure reproduce against a fresh `setup.ts`-based install, or
      only against a stale/plain one? A failure that only reproduces on a
      plain `deno install -g` is this skill's territory, not a genuine
      resolver regression.

## Out of scope — do not do these

- The resolver logic that rewrites a SERVED project's own bare specifiers
  once a command is actually running (`resolveReplacement`,
  `cliLoaderHasNoRealLocalAnswer`, `resolvesIntoCliOwnSourceTree`) — that's
  a different, already-running-command concern; `deno-workspace-link-pitfalls`
  covers the general `@deno/loader` `Workspace` gotchas it also hits.
- Deciding whether a command's own dispatch SHOULD be lazy/dynamically
  imported — that's `cli-command-architecture`'s territory (the command-tree
  wiring convention itself); this skill only covers the downstream
  install-time consequence of that pattern already being in place.
- General `deno install -g` behavior for a package that ISN'T `@zanix/cli`
  or doesn't share its lazy-dispatch pattern — the shim-config gap is
  universal to `deno install -g`, but the lockfile gap specifically depends
  on a package having dependencies reachable only through a dynamic
  `import()` the static graph never sees.

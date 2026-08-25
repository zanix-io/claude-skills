---
name: cli-artifact-generators
description: Architecture and conventions for zanix generate's per-artifact generators — module layout, the plan<Name> pure-planner pattern, the parser/renderer split for field-based DSLs, and the doc-sync rule. Use when adding a new generator, changing an existing one's output shape, or reviewing a generate/<artifact>/ module.
---

This skill covers `zanix generate <artifact> <name>` — adding one artifact to
an already-existing project. For `zanix new` (bootstrapping a whole project),
see `cli-scaffold-assembly`; both share the `plan<Name>` functions this skill
describes, so a generator's shape can never drift between the two commands. For
command-tree wiring (`Commander`/`mountGroup`), see `cli-command-architecture`.
File:line references point at `~/Documents/Development/ZanixLibraries/cli` —
read the real code there before assuming this summary is still accurate.

## Golden rule (token savings)

- **Evidence means a targeted read, not a full-file tour.** Confirming one
  specific fact (a function's real signature, whether a type is exported)
  needs a `grep`/`deno doc <symbol>`/a few lines around the definition — not
  reading the whole source file top to bottom. Read a whole file only when you
  genuinely need its overall shape, e.g. picking a sibling generator as the
  template for a new one.
- **Copy a sibling generator's shape, don't re-derive it.** The module layout,
  the `plan<Name>` split, the registration pattern — an existing
  `<artifact>/command.ts` closest to what you're building already gets all of
  this right. Adapt it; don't re-read `cli-command-architecture` and
  re-reason the pattern from first principles for every new generator.
- **Trust this skill's Ecosystem Conventions table** for artifact types you're
  not touching — don't re-verify every row against real source before
  starting. Verify only the row(s) actually relevant to the artifact you're
  building right now.
- See `cli-generator-testing`'s own Golden rule for the Validation step
  specifically — running the full test suite is expensive and belongs at
  defined checkpoints, not after every file written.

## The standing workflow for any generator feature

Every generator — existing and future — follows the same discipline:

1. **Evidence** — read real production usage of the artifact being generated
   (real repos, real decorator signatures, real published dependency APIs)
   before writing any template. Don't assume a shape; verify it.
2. **Decisions** — where genuine design choices exist (input mechanism, file
   granularity), confirm with the user rather than guessing.
3. **Plan** — write down what's being built and why before implementing.
4. **Implementation** — build the generator.
5. **Validation** — `deno check` the generated *output* against the real,
   currently-published version of whatever it imports (not an assumed API
   shape — see `cli-dependency-compatibility`); 100% branch/function coverage
   on new code; full test suite green; `deno lint`/`deno fmt --check` clean
   (see `cli-generator-testing`).
6. **Docs** — update `docs/generate.md`/`docs/generate-space.md` (every
   generator, every time — see below), and **also `docs/engineering.md`**
   whenever the change introduces a new standing convention (a module-layout
   exception, a new field-input mechanism, a new scaffolding-ownership rule)
   rather than just another same-shape artifact. `docs/generate.md` alone
   getting updated while `docs/engineering.md` doesn't is a real drift this
   ecosystem has actually hit — a generator can ship fully documented for
   users while the internal architecture doc silently falls behind; check
   both, don't assume updating one covers the other.

## Ground truth: artifact concept → decorator → base class → library

Verified against real production repos, not assumed — re-verify before relying
on it if enough time has passed that it might have drifted:

| Concept              | Decorator                                                         | Base class                                                                                              | Library     |
| --------------------- | ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------- | ----------- |
| REST handler          | `Controller({prefix,Interactor})` + `Get/Post/Patch/Delete/Put`    | `ZanixController<I>`                                                                                        | server      |
| GraphQL handler        | `Resolver({prefix})` + `Query`/`Mutation`                          | `ZanixResolver`                                                                                             | server      |
| Socket handler         | `Socket(route)`                                                    | `ZanixWebSocket`                                                                                             | server      |
| SSR handler            | `SsrController({prefix})` + same method decorators as REST         | `ZanixSsrController`                                                                                         | server      |
| Interactor             | `Interactor({Connector,Provider})` (optional)                      | `ZanixInteractor<T>`                                                                                         | server      |
| Provider/repository    | `Provider(type?)`                                                  | `ZanixProvider<T>`                                                                                           | server      |
| Connector shell        | `Connector({slot?})`                                               | `ZanixConnector` / `ZanixDatabaseConnector` / `ZanixCacheConnector` (by `--slot`)                            | server      |
| Queue consumer         | `Subscriber(route\|{queue,rto,Interactor})`                        | `ZanixSubscriber`                                                                                            | asyncmq     |
| Jobs                   | `registerCronJob`/`registerJob` (plain functions, not decorators)  | —                                                                                                            | asyncmq     |
| DLQ processor          | `registerDLQProcessor` (plain function)                            | —                                                                                                            | asyncmq     |
| DB connector (default) | —                                                                  | `ZanixMongoConnector`, `registerModel<Attrs>({name,definition,options,extensions:{seeders},callback})`       | datamaster  |
| RTO/DTO field          | `@zanix/validator` decorators (`IsString`, `IsEmail`, ...)          | `BaseRTO`                                                                                                     | validator (alias of `@zanix/utils/validator` — see `cli-dependency-compatibility`) |
| Comet                  | `'use comet'` directive + `defineComet(Component, import.meta.url)` | —                                                                                                             | space       |
| Page                   | `@Page()`                                                          | `SpacePageController`                                                                                        | space       |
| Middleware (guard/pipe/interceptor) | `defineMiddlewareDecorator(kind, fn)`                | —                                                                                                             | server      |

`@zanix/datamaster` does **not** export repository classes — those are app-code
`ZanixProvider` subclasses, which is why `repository` generates one alongside
its `model.defs.ts` rather than importing a pre-built class. `@zanix/auth` has
no per-domain file shape of its own — it's "a decorator you import," not
something any generator here scaffolds.

## Module layout: one self-contained folder per artifact

```
src/commands/generate/
  main.ts              -- creates the `generate` sub-Commander, iterates registry.ts, nothing else
  registry.ts           -- Array<(cwd: Commander) => void> of every register*Command function
  shared/
    project.ts           -- assertProjectType/getCurrentProjectType (generate-only)
  <artifact>/
    command.ts            -- register<Name>Command + the action + the pure plan<Name> function
    template.ts            -- (or renderer.ts for rto — see below) the string-builder template(s)
```

**Adding generator #N** means: create `<artifact>/command.ts` (+
`template.ts`), add one entry to `registry.ts`. `main.ts` never changes — it
just iterates the registry.

**The `plan<Name>` convention**: each `<artifact>/command.ts` also exports a
*pure* `plan<Name>(...)` function — no `Commander`, no `assertProjectType`, no
logging, just "what files (and what side-effect files, e.g. a shared helper
written once) does generating this artifact need." Both
`generate<Name>Action` (the CLI action) and `cli-scaffold-assembly`'s project-tree
leaves call the exact same `plan<Name>` — neither can drift from the other by
construction, since there's only one function that knows the shape.

## Templates are inline `.ts` string-builders, never separate `.tpl` files

`zanix build` bundles a whole project into a single output file for its users
— a runtime file read relative to `import.meta.url` would resolve against that
bundle's own location, not the original source layout, for any project
(including `cli` itself, if it were ever built this way) that gets bundled.
Staying inline avoids depending on `cli` never being run through its own
`build` command for some other purpose. **Watch for**: if a `template.ts`/
`renderer.ts` grows to hundreds of lines, reconsider whether it should read
from an external file instead — deferred until it's an actual problem, not
designed for pre-emptively.

## Field Model Convention: DSL string → Parser → structured model → Renderer

Where a generator needs real per-field input (currently only `rto`, via
`--field name:type[?][[]]`), the pipeline is strictly:

- `<artifact>/parser.ts` — pure DSL-string parsing. Zero decorator/codegen
  knowledge. Owns its own literal list of supported type names (not derived
  from the renderer's mapping table — the two are kept in sync by hand, on
  purpose, since the parser must never import renderer-only knowledge).
- `<artifact>/renderer.ts` — consumes the structured model, owns all
  decorator/TS-type/codegen knowledge (`rto/renderer.ts`'s `FIELD_TYPE_INFO`:
  which `@zanix/validator` decorator, which TS type, whether the import is
  local, whether the decorator even accepts `expose` — verified against the
  real published `@zanix/validator` API via `deno check`, not assumed).
  **Before setting `localImport: true` for a new field type, check whether
  `@zanix/utils`'s validator module already has — or genuinely should
  have — a real decorator for it first (see `utils-builder`'s own
  "Adding a new validation decorator" workflow).** `localImport: true`
  means "hand-template a local one-off validator into every generated
  project" — a real, legitimate escape hatch when nothing generic exists,
  but not a default reached for without checking. Two confirmed real
  mistakes from reaching for it without that check: `objectid` hand-templated
  its own `OBJECTID_REGEX` when `@zanix/utils` already had the identical
  `IsUUID`-shaped pattern to follow (fixed — promoted to a real
  `IsObjectID`, `localImport` removed); `permission` hand-templated a
  regex (`/^[A-Za-z-]+:[A-Za-z-]+$/`) that didn't even match real
  production usage anywhere in the ecosystem (`@zanix/admin`'s own
  hierarchical `zanix:admin:triggers`-shaped permissions, and bare
  `'admin'`, both real and both rejected by it) — removed entirely, no
  replacement, because no converged format exists to validate against.
  **If the validation shape is genuinely novel** (not a generic,
  ecosystem-wide concept like an ObjectId/UUID/email), grep real usage
  across the other 11 repos before assuming any specific format is
  correct — a plausible-looking regex invented without checking real
  usage is worse than no validation at all.

This split exists so future DSL growth (a constraint modifier, a new field
type) only touches the parser's syntax layer or the renderer's codegen layer,
never forces a change to both.

## Shared vs. cross-cutting utilities: where new generate-adjacent logic belongs

- **Genuinely generate-only** (encodes a generate-specific precondition or
  concept) → `generate/shared/`. Example: `assertProjectType` — "this
  operation requires an already-existing project of type X" only makes sense
  for incrementally adding to a project, never for `zanix new` (which is
  creating the project).
- **Horizontal, multi-consumer, or plausibly needed outside `generate/`** →
  stays in `src/utils/`. Example: `casing.ts` (`toKebabCase`/`toPascalCase` —
  zero generate-domain coupling), `projects/creation.ts`
  (`createFilesAndFolders`/`ensureConstant` — used by both `generate/`'s and
  `new/`'s actions).

Don't create a shared file just to match a folder-shape convention —
`generate/shared/` should only ever contain what's genuinely generate-specific.

## Public documentation must move in the same change

`README.md` and `docs/{generate,generate-space}.md` are the user-facing
counterpart to this skill — command reference, options tables, and verified
real-output examples for every artifact, kept accurate against the real
`command.ts`/`template.ts` source rather than described from memory. **Adding
generator #N to `registry.ts` means adding its row + its own example section to
the matching doc in the same change** — an undocumented generator is exactly
the kind of drift this convention exists to prevent. Keep `CHANGELOG.md` and
`deno.jsonc`'s `version` in sync too (see `release-management`). This is the
user-facing half of the Docs step above — `docs/engineering.md` is the
separate, internal half, needed only when the change is architectural, not
for every generator.

## Current generator surface (verify against `docs/generate.md`/`docs/generate-space.md` before relying on the exact shape)

Backend (`server`/`space-server` projects, generate relative to `src/server/`):
`seeder`, `repository`, `handler` (`--type rest|graphql|socket|ssr`), `rto`
(`--field name:type[?][[]]`, repeatable), `connector` (`--slot
database|cache:<subtype>`), `interactor`, `job` (`--cron`), `dlqprocessor`
(`--process-type`, `--schedule`, both required), `subscriber` (`--queue`),
`middleware` (`--kind guard|pipe|interceptor`, required — three equally
common concerns with no natural default to guess at, same reasoning as
`dlqprocessor`'s required options).

`middleware` generates `middlewares/<name>.<kind>.ts`, a shell built directly
on `defineMiddlewareDecorator` — the one real primitive behind all three
kinds in `@zanix/server` (`Guard`/`Pipe`/`Interceptor` sugar decorators are
each a one-line wrapper around it). `middleware` **is** seeded by `zanix new`
today, for every non-`library` project type (`server`/`space`/`space-server`/
`app`): `main.ts`'s own `MIDDLEWARES_RECIPE` calls `zanix generate
middleware`'s own `planMiddleware('example', 'Example', 'pipe'|'interceptor',
folder)` directly, producing `src/shared/middlewares/example.pipe.ts` and
`example.interceptor.ts` — a separate ad-hoc recipe in `main.ts`, not a typed
`ZanixServerSrcTree` leaf. `dlqprocessor`/`subscriber` remain the real
"generate-only, no `zanix new` seeding" examples: no `@zanix/utils`-published
tree type declares a folder for either yet. Don't assume every generator in
this table also has a matching `zanix new` seeding leaf — `cli-scaffold-assembly`
only seeds what a project type's own recipe declares, and a generator can
legitimately exist without one, same as `dlqprocessor`/`subscriber` today.

Frontend (`space`/`space-server` projects, generate relative to `src/space/`):
`comet`, `component` (no framework-discovery convention — a plain
presentational shell meant to be imported by hand into a page/layout/another
component's JSX tree, not Recipe-wired into `zanix new space`/`spacecraft`
yet), `page` (`--route-path`, supports dynamic segments like
`'products/[id]'`), `layout`, `error`, `loading` (React-only — rejected at
route-registration time under `renderer: 'preact'`), `not-found` (no name
argument — a single, whole-app file at the routes root).

Every generator: only runs inside the right project type (erroring out
otherwise); never overwrites an existing file (safe to re-run); accepts an
optional trailing `root` argument; supports opt-in `--verify` (see
`cli-dependency-compatibility`).

## Checklist before adding or changing a generator

- [ ] Does `Evidence` back the shape (real repos/decorator signatures/published
      APIs), not an assumption?
- [ ] Does `<artifact>/command.ts` export a pure `plan<Name>` function that both
      the CLI action and `cli-scaffold-assembly`'s tree leaves can call, rather
      than duplicating the file list in two places?
- [ ] Is the template an inline `.ts` string-builder, not a file read at
      runtime?
- [ ] Does this generator need per-field input? If so, is DSL parsing
      (`parser.ts`) kept fully separate from codegen knowledge (`renderer.ts`)?
- [ ] Is new generate-adjacent logic placed by the genuinely-generate-only vs.
      horizontal-utility criterion, not by folder-shape habit?
- [ ] Did `docs/generate.md`/`docs/generate-space.md` get a new row + example in
      the same change as the new generator?
- [ ] Before setting `localImport: true` on a new field type, was
      `@zanix/utils`'s validator module (and real usage across the other
      11 repos, if the shape is novel) actually checked first — not
      assumed there's nothing to reuse?
- [ ] Does this change introduce a new standing convention (not just another
      same-shape artifact)? If so, did `docs/engineering.md` get updated too —
      not just the user-facing doc?
- [ ] Does the generated *output* still `deno check` clean against real,
      currently-published dependency versions (see `cli-dependency-compatibility`)?

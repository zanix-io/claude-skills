---
name: cli-scaffold-assembly
description: How zanix new assembles a whole project tree — the Recipe/Assembler abstraction, presets, and the ownership boundary between generator-backed content and the one legitimate JSR-fetch exception. Use when adding a new project type, a new preset, or changing what a project type's scaffold seeds.
---

This skill covers `zanix new <type>` — bootstrapping a whole new project. For
`zanix generate` (adding one artifact to an existing project), see
`cli-artifact-generators`; both share the `plan<Name>` functions that skill
describes, which is what keeps a project type's scaffold from ever drifting away
from what `generate` produces for the same artifact. File:line references point
at `~/Documents/Development/ZanixLibraries/cli` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- Confirm the Recipe/preset mechanics by reading `recipe.ts`/`presets.ts` once,
  not once per project type touched — the mechanism (`resolveRecipe`,
  `assembleScaffold`'s append semantics) is identical across `server.ts`/
  `space.ts`/`app.ts`; only each file's own `*_RECIPE_BASE` content differs, and
  that's the only part worth a fresh read per project type.
- Don't regenerate a whole project tree to confirm a leaf's shape when a
  targeted `grep` for the leaf's `plan<Name>` call already answers it — reserve
  a real `zanix new` run for the actual Validation step, not for exploration.

## Two mechanisms, and which one new content should go through

- **`zanix generate <artifact> <name>`** — parameterized, evidence-verified,
  tested string-builder functions (`cli-artifact-generators`). Adds _one_
  artifact to an _already-existing_ project.
- **`zanix new <type>`** — bootstraps a _whole new project_, assembling a folder
  tree (`src/commands/new/lib/tree/`) whose leaf nodes are either (a)
  locally-generated content calling the exact same `generate/` `plan<Name>`
  functions, or (b) a static example file fetched at runtime from another
  library's own `src/templates/` folder via JSR
  (`getZanixTemplateContent`/`ZanixTree`,
  `src/commands/new/lib/tree/{templates,base-tree,tree,info}.ts`).

## Ownership boundary: generator-backed content, never a hand-maintained static copy

**For any artifact type that already has (or could have) a `zanix generate`
generator, `cli`'s own generator is the single source of truth.** Never fetch a
separately hand-maintained static copy from another library's `src/templates/`
for that shape. This isn't a style preference — a hand-maintained static copy
produces two concrete failure modes:

1. **Drift** — the static example silently falls behind what the real generator
   produces, so a freshly scaffolded project's example files look thinner or
   shaped differently than what `zanix generate` would write for the same
   artifact today.
2. **A missing or misplaced static file can silently scaffold empty output**,
   with nothing to cross-check the static copy against the generator's real
   output.

Every leaf in `src/commands/new/lib/tree/projects/server.ts` (and `space.ts`)
generates locally today, calling `cli`'s own template functions directly with a
placeholder name (`'Example'`/`'ExampleCounter'`) — no JSR fetch for any backend
or frontend artifact that has a `zanix generate` counterpart.

**One real exception to "every leaf calls its artifact's `plan<Name>`
function"**: `server.ts`'s `repositories` leaf deliberately calls
`modelDefsTemplate('Example', 'example')` directly instead of `planRepository`.
A full repository (what `planRepository` plans) is `entity.provider.ts` +
`model.defs.ts` together; the scaffold wants only a standalone `model.defs.ts`
example — a lighter shape by design, not a static hand-maintained copy, still
generator-backed (`generate/repository/template.ts`'s own `modelDefsTemplate`).
See that leaf's own comment in `server.ts` for the full reasoning.

**What still legitimately fetches from another library's `src/templates/`**:
only `@zanix/utils`'s own generic, non-API-coupled project skeleton
(README/LICENSE/CHANGELOG/generic example files) for `library` projects —
there's no generator to defer to (a library isn't a single artifact type), and
it isn't `cli`-specific domain knowledge either way. This is not a closed list
forever: check any _new_ per-artifact-type static example a library might add
later against the same criterion before accepting it — if it's a generate-unit
shape, it belongs in `cli`, not fetched.

## The Recipe/Assembler abstraction

A project type's scaffold is a declarative array, not a hand-written imperative
block per leaf:

- `ScaffoldRecipeEntry<Tree>` (`{leaf, plan}`) — pairs a tree location with the
  `plan<Name>` function (or static content) that fills it.
- `assembleScaffold(tree, recipe)` (`commands/new/lib/tree/recipe.ts`) — turns a
  project type's leaves into that one declarative array, and also collects each
  entry's optional `sideEffects` (a `plan<Name>`-style side effect, like `rto`'s
  `ensureConstants` or `seeder`'s `ensureHelper`) and hands them back to the
  caller — so a future leaf's side effect is picked up automatically instead of
  needing a matching call added by hand elsewhere.
- **`assembleScaffold` appends onto a leaf's existing `templates.base`, never
  replaces it**
  (`leaf.templates = { base: [...leaf.templates.base,
  ...plan.files] }`). This
  matters once a recipe's `leaf` resolves to a node that already carries real
  content (a project's root `README.md`/`LICENSE`) — replace semantics would
  silently wipe it.

`server.ts`'s `SERVER_RECIPE_BASE` and `space.ts`'s `SPACE_RECIPE_BASE` are the
real recipes for those two project types.

## When a new `generate` artifact has no typed leaf to hang a Recipe entry on

Not every artifact `zanix generate` can produce has somewhere to go in
`zanix
new`'s own tree — the leaf types a project type's tree can declare
(`ZanixSpaceSrcTree`, etc.) are published from `@zanix/utils`, outside `cli`'s
own control, and only declare the leaves that already exist there (`routes`,
`comets` for `ZanixSpaceSrcTree` — confirmed real by reading
`@zanix/utils/src/typings/zanix.ts` directly, not assumed). **Before deciding a
new generator "should" wire into a project type's Recipe (step 3 of
`cli-artifact-generators`'s own workflow), confirm a typed leaf for it actually
exists** — don't assume its absence is an oversight to fix inside `cli`. If none
exists, wiring it in would require a `@zanix/utils` release first, which is out
of scope for a `cli`-only change: document the artifact as `generate`-only (no
`zanix new` seeding) until that leaf exists, the same precedent already set for
`component` (see `docs/generate.md`) — `middleware` used to be cited here too,
but it's since been wired into `main.ts`'s `MIDDLEWARES_RECIPE` and is now
seeded by every non-`library` project type, so it's no longer a live example of
this case. This is a real, recurring case, not a one-off — check for it
explicitly instead of treating every new generator as automatically
Recipe-eligible.

## Presets

`zanix new --template <preset>` (default `'base'`) resolves through:

```
project type → preset → Recipe → assembleScaffold()
```

- `commands/new/lib/tree/presets.ts` — `PresetName` and `KNOWN_PRESETS`.
  `assertKnownPreset(preset)` throws a plain `Error` for anything not in the
  list, called first thing inside `getZnxFolderTree`, before any tree is built
  for _any_ project type.
- `ScaffoldRecipeRegistry<Tree>` (`Record<string, ScaffoldRecipeEntry<Tree>[]>`)
  - `resolveRecipe(registry, preset)` — throws the same kind of `Error`, scoped
    to one project type's own registry. `SERVER_RECIPES`/`APP_RECIPES` are
    plain top-level `const` objects; `space`'s registry is instead a function,
    `getSpaceRecipes(theme, renderer)` (`space.ts`) — built fresh per call
    because `welcome`'s own `routes` leaf needs both threaded through (its
    copy adapts to `theme === 'astronaut'`, its `@zanix/space-ui` import
    resolves against `renderer`), unlike `SERVER_RECIPES`, which has nothing
    on the server side that varies with either. `APP_RECIPES` is still just
    `{ base: APP_RECIPE_BASE }` — a single root `mod.ts` entry has nothing
    else to vary on yet. `SERVER_RECIPES` and `getSpaceRecipes(...)` have
    grown real content beyond `base` — see Presets below.
- `getServerSrcTree`/`getSpaceSrcTree` key their module-level tree cache on
  `` `${startingPoint}::${preset}` ``, not just `startingPoint` — the same root
  requested with two different presets must never return a stale tree built for
  the first one.
- Extending this for a real preset #2 touches none of `assembleScaffold`,
  `resolveRecipe`, or any generator's own `command.ts` — only
  `PresetName`/`KNOWN_PRESETS` (widen) and the owning project type's own
  registry (add one entry) change.

**Designing a preset's actual _content_ is a separate, evidence-first product
decision from this infrastructure.** `'base'` is no longer the only preset
with real content: `space`/`spacecraft` also support `--template welcome`
(a real welcome landing page, composed from `@zanix/space-ui`'s `Link`) and
the mutually-exclusive `--template population`/`population-lang` (a real,
working i18n/population reference — guards, `messagesDir`, real catalogs).
`SERVER_RECIPES.welcome`/`.population`/`.population-lang` are deliberate
aliases for `SERVER_RECIPE_BASE` (same array reference) so `spacecraft`'s
server half — which has no landing-page/i18n concept of its own — still
resolves for the same `--template` value; they add no server-specific
content. Visual identity (`--theme default|astronaut`) is a SEPARATE axis
from `--template`, deliberately never routed through the Recipe/preset
mechanism at all — see `themes.ts`'s own doc; it never reaches
`SERVER_RECIPES`/`getSpaceRecipes` and has nothing to do with which preset is
active. Don't treat the existence of the registry mechanism as license to
invent preset content speculatively — evidence-first still applies to any
NEW preset beyond these.

## `library` is the one exception to the Recipe mechanism, for a narrower reason than the JSR-fetch boundary above

`library`'s only artifact, `src/modules/mod.ts`, is a static, generic
placeholder fetched declaratively from `@zanix/utils`'s own `src/templates/` —
not content a `plan<Name>` call generates locally, because a library's whole
point is user-authored content with no fixed shape the CLI could know ahead of
time (unlike `server`'s example handler, which has a real, generator-verified
shape). `getLibrarySrcTree` still gets the same defense-in-depth validation
`resolveRecipe` gives the other project types: it calls `assertKnownPreset` a
second time directly, against its own `LIBRARY_KNOWN_PRESETS` list — only its
_content generation_ stays on the declarative JSR-fetch mechanism, not its
preset validation.

## Project types (verify against `docs/new.md` before relying on the exact shape)

| Type         | Creates                                                                                                | Notable options                                                                              |
| ------------ | ------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------- |
| `app`        | A `defineZanixApp()` manifest package (`mod.ts`) — no `src/server`/`src/space`/`src/modules` subfolder | —                                                                                              |
| `space`      | A `@zanix/space` frontend app (`src/space/`)                                                           | `--renderer react\|preact`, `--icons`, `--theme default\|astronaut`, `--pages`, `--template`   |
| `server`     | A backend server (`src/server/`), plus root `mod.ts`/`worker.ts` entrypoints                           | `--template`                                                                                  |
| `spacecraft` | `space` + `server` combined under the same `src/`                                                      | Same as `space` — `--theme`/`--icons`/`--pages` never affect the server half                  |
| `library`    | A reusable library (`src/modules/`)                                                                    | —                                                                                              |

Every type shares `-t/--template` (`base` by default; `space`/`spacecraft`
also accept `welcome`/`population`/`population-lang` — see Presets below),
`--no-prepare` (skip the automatic `zanix prepare -g -e` step), and
`--verify` (see `cli-dependency-compatibility`). `zanix new` never
overwrites — every generated project is a fresh directory.

## Checklist before adding a new project type or changing a scaffold's shape

- [ ] Does every leaf that has a `zanix generate` counterpart call that
      artifact's real `plan<Name>` function — never a separately maintained
      static copy?
- [ ] For a NEW generator being considered for Recipe wiring: does a typed leaf
      for it actually exist in the project type's own tree type
      (`@zanix/utils`-published), or is "not seeded by `zanix new` yet" the
      correct, documented state until that leaf ships?
- [ ] Is a genuinely new, non-generator-backed static asset actually
      non-API-coupled and generator-unit-shaped-nothing (the `library`/
      `@zanix/utils` exception), or should it really be a generator?
- [ ] Does the new/changed leaf go through `assembleScaffold`'s append
      semantics, so it can't silently wipe pre-existing content on that node?
- [ ] Does a new project type get its own `*_RECIPE_BASE`/`*_RECIPES` registry
      and `assertKnownPreset` validation, consistent with the other four?
- [ ] Is this change to _infrastructure_ (the Recipe/Assembler/preset mechanism)
      kept separate from any change to _preset content_ — the latter is its own
      product decision?

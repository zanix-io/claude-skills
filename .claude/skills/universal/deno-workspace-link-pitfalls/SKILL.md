---
name: deno-workspace-link-pitfalls
description: Real, confirmed-via-repro footguns in TEMPORARILY linking an unpublished/pre-release sibling package via a raw relative-path `deno.jsonc` `imports` override (not the `links` array) — `scopes` alias-name collisions, the exact prefix depth a `scopes` key needs to actually take effect, per-subpath override coverage gaps, and a raw local-path override silently auto-linking unrelated sibling directories too. Applies to any Deno monorepo/workspace testing a fix across sibling repos before publish — not Zanix-architecture-specific, though every confirmed case here happened in the Zanix ecosystem. Split out of `deno-lazy-dependency-pattern` (2026-08-25) once that skill's own TEMP-linking content grew into a second, genuinely separate responsibility from the lazy-dependency pattern itself. Use whenever adding, debugging, or reviewing a TEMP local-path override plus its paired `scopes` block, or when a fix that should be live via such a link doesn't seem to be taking effect.
---

Grounded in a real, same-day investigation (2026-08-25) across `@zanix/admin`, `@zanix/asyncmq`,
`@zanix/datamaster`, `@zanix/server`, `@zanix/space`, `@zanix/cli`, and `console` — every package in
that day's fix required linking at least one unpublished sibling checkout to test a fix before
publish, and every one of the four footguns below was hit for real, not hypothesized, in the course
of doing that. Originally captured as part of `deno-lazy-dependency-pattern`; split out once that
skill's own TEMP-linking content grew into a genuinely separate responsibility (testing a
cross-repo link, not designing a lazy-loaded dependency) — see that skill for the underlying
over-materialization problem this linking technique is usually deployed to verify a fix for.

## Golden rule (token savings)

- **A TEMP local-path override (`"@zanix/pkg": "../pkg/mod.ts"` in `imports`, not the `links`
  array) is the standard way to exercise an unpublished sibling package's real, unreleased fix
  before it publishes** — a raw relative path wins over whatever version range a package declares
  internally, because Deno resolves the whole graph for one run against a single governing config.
  Every footgun below is about this technique specifically, not about `links` (which respects
  semver ranges and can silently fall back to a real published version that satisfies a site's own
  declared range instead of forcing the local checkout — not covered here since it's a different
  failure mode for this exact workflow). **One real, confirmed `links`-specific case exists outside
  this skill's scope**: `"links"` only takes effect at the workspace ROOT `deno.json` — a consuming
  project's own `"links": ["../space"]` is silently ignored whenever it's reached through a
  dynamically-imported entry file belonging to a DIFFERENT local-checkout tool (e.g. `@zanix/cli`
  itself run via `deno run -A ../cli/mod.ts space dev`, which resolves the imported project's bare
  specifiers against `cli`'s own `deno.jsonc`, not the project's). **STALE CLAIM, confirmed wrong
  2026-09-02**: this used to say `@zanix/cli`'s own `warn-unhonored-links.ts`
  (`warnUnhonoredProjectLinks`) ships as a real, wired-in mitigation for this — a real `find`/`grep`
  against the actual `cli` checkout (`src/`) turned up NEITHER the file NOR the function anywhere.
  Either it was removed, never actually shipped despite being documented here, or this entry was
  wrong from the start — don't cite it as real until re-confirmed against the checkout. Check
  `@zanix/cli`'s own `docs/space.md` for whatever the CURRENT real behavior is, and re-verify before
  trusting either this entry or that doc blindly. **A real, reproduced instance of this exact
  footgun** (same day): running `zanix space dev` via a LOCAL `cli` checkout entrypoint resolved a
  consuming project's own npm dependency's CSS asset (`graphiql/graphiql.css`) under the `cli`
  checkout's OWN `node_modules`, not the served project's — reproduced with no warning of any kind
  surfaced, consistent with the mitigation not actually existing. See `space-styling-and-theming`'s
  own entry on this for the full incident (including a separate, DISPROVEN hypothesis that first
  blamed `@zanix/space`'s own dev-engine CSS handling, before this linking mechanism was confirmed
  as the real, much more likely cause via a clean repro against a non-linked project).
- **Every one of these footguns shares the same shape**: the resolution graph LOOKS correctly
  linked (a `deno info`/`deno run` warning cites the right local path for the package you meant to
  link), so "is the package actually linked" reads as a settled question — but ONE specific file,
  subpath, or bare specifier within that supposedly-linked graph is still silently resolving to
  something else (the real published version, an unrelated sibling's own WIP, a different local
  checkout entirely). The resulting symptom looks exactly like "the fix doesn't work" or "some
  unrelated layer is broken," burning real investigation time before the actual cause (an
  incomplete override, not a broken resolver) is found. **The general tell across all four**: before
  trusting any theory about `scopes`, referrer-threading, or per-package config discovery being
  broken, check what a `deno run` warning or `deno info --json` `modules` entry says the SPECIFIC
  file/symbol involved in the failure actually resolved to — not the bare package's own root.

## A paired `scopes` block is NOT always required — it solves ONE specific problem, not "does the link resolve at all"

Every footgun below assumes a TEMP override is paired with a `scopes` block redirecting the linked
package's own bare specifiers. **That pairing is conditional, not universal — confirmed by testing
both the positive and negative case directly, not assumed from the presence of `scopes` in earlier
real fixes.** Two things a `scopes` block does NOT do, each confirmed via a real, isolated repro:

- **It is not needed just to make the linked package's own bare specifiers resolve.** Deno's own
  nearest-config-file discovery already resolves a file's bare specifiers against ITS OWN nearest
  ancestor `deno.json(c)` — a raw relative-path override into a package's real entry file, with NO
  paired `scopes` at all, resolves that file's own internal `typings/`/`modules/`/`@zanix/server`-
  style aliases correctly on its own. Confirmed twice, independently: `@zanix/asyncmq`'s own real
  `deno.jsonc` carries an explicit audit note that removing its `../server/` scope made zero
  difference (full suite still 90/90); and a from-scratch isolated repro linking `@zanix/datamaster/
  dlq`'s real entry file with no `scopes` block at all installed the identical, correct dependency
  set (`npm:mongoose` only) as the same repro WITH a `scopes` block added.
- **A `scopes` entry does not force materialization just by being declared, unlike a top-level
  `imports` entry.** The golden rule's Trigger 1 (an unused `npm:`-literal alias in the CURRENT
  project's own top-level `imports` map materializes regardless of reachability) does NOT extend to
  `scopes` — confirmed via a real repro: an `npm:left-pad@1.3.0` literal placed inside a `scopes`
  block, under a prefix nothing in the actual module graph reaches, was never installed. A `scopes`
  entry follows the SAME reachability rule as Trigger 2 (a dependency's own `imports` map) — declared
  ≠ materialized, only ACTUALLY RESOLVED ≠ materialized.

**So when IS a `scopes` block actually needed?** Only for the one real problem it solves: preventing
a DUPLICATE MODULE INSTANCE when the SAME logical dependency is reachable via two different paths in
the same overall graph — once through the rest of the consuming project's own real, published
specifier (e.g. `jsr:@zanix/utils@^3.1.2/logger`, used everywhere else in that project), and once
through the newly-linked package's OWN internal resolution of that same logical dependency (which,
left alone, would resolve to a second, different instance of the same class/symbol — breaking
`spy()`/`instanceof` checks across the two). If nothing else in the consuming project's own graph
already resolves the linked package's shared dependencies via a different path, there is no
duplicate-instance risk to guard against, and no `scopes` block is needed at all — confirmed by the
`@zanix/asyncmq`/`datamaster` isolated-repro case above, where nothing else in either graph competed
for the same specifier. Check for this BEFORE adding a `scopes` block defensively: does anything
else already reachable from the SAME `deno check`/`deno test`/`deno run` invocation resolve the
linked package's own shared dependency via a different specifier? If not, skip the `scopes` block
and confirm with a real repro that omitting it doesn't change the materialization result.

## Footgun 1 — a `scopes` entry reusing the same alias NAME as a top-level `imports` entry is spec-correct, but a bundler/dev-server bridge on top of it might not thread the referrer needed to resolve the collision

A consuming project linking an unpublished/pre-release dependency via a raw relative-path `imports`
entry, paired with a `scopes` block redirecting that dependency's OWN bare specifiers to its real
source tree (e.g. the linked dependency's own `utils/` alias needing to resolve to ITS `./src/utils/`,
not the consuming project's identically-named `utils/` alias) — not specific to any one package,
confirmed via a real, isolated repro.

**A `scopes` entry that reuses the same alias NAME as a top-level `imports` entry (e.g. both
declare `utils/`, one at top level for the consuming project's own `./src/utils/`, the other
nested under `scopes["../dependency/"]` for the dependency's own `./src/utils/`) is CORRECT and
unambiguous at the plain Deno-resolution layer (`deno info --json`, `deno check`, `deno test`) —
per-scope resolution is real, spec-compliant WHATWG Import Maps behavior, matched against the
REFERRER (the importing file's own path), not the specifier alone. But a bundler/dev-server bridge
built on top of that resolution layer can still get it wrong if IT fails to thread a real referrer
through its own call into the resolver** — confirmed as the actual, sole root cause of a real
`zanix space dev` crash (`console`, a real consumer project, 2026-08-25): `Failed to load url
/src/utils/targets.ts (resolved id: .../console/src/utils/targets.ts) in .../server/mod.ts` —
`@zanix/space`'s own SSR dev-server Vite bridge (`bare-specifier-resolve.ts`'s
`canonicalBareSpecifierResolvePlugin`, registered ahead of `@deno/vite-plugin`'s own `deno()` and
always short-circuiting) called `@deno/loader`'s `Loader.resolveSync(id, referrer, mode)` with
`referrer` hardcoded to `undefined` for every importer, which makes ANY `scopes` entry
structurally unreachable regardless of collision — it silently fell back to the top-level
`imports["utils/"]` entry for a bare `utils/targets.ts` import inside the LINKED dependency's own
`mod.ts`, resolving into the consuming project's own (wrong, and in that case nonexistent)
directory instead. Fixed at the actual bug — `@zanix/space` now computes a real referrer URL from
the Rollup `resolveId(source, importer, options)` hook's own `importer` argument (mirroring
`@deno/vite-plugin`'s own already-correct, internal-only `memberReferrerUrl` logic) and threads it
through, so a `scopes` override wins over a same-named top-level entry exactly as the spec
requires. This was a real, fixable bug in `@zanix/space`'s own bridge code, not a limitation of
`@deno/loader`/`@deno/vite-plugin` (`@deno/loader`'s public `resolveSync` signature has always
accepted a `referrer` for exactly this reason, and `@deno/vite-plugin`'s own primary resolver
already computes one correctly — the bug was `@zanix/space` bypassing that correct path with its
own referrer-less shortcut). **Confirmed via a real, live `deno run -A ../cli/mod.ts space dev`
repro from `console` (2026-08-25) that the fix genuinely works** — but "once this project links a
`@zanix/space` build carrying that fix" turned out to be the load-bearing, easy-to-get-wrong
precondition: `console/deno.json`'s own `@zanix/space` override only ever covered the bare `.` and
`/react` exports, and `@zanix/cli`'s own `commands/space/dev/command.ts` reaches the dev bridge
through the SEPARATE `@zanix/space/dev` subpath, which neither `console/deno.json` nor
`cli/deno.jsonc` had overridden locally — so the crash kept reproducing identically for hours,
looking exactly like the fix didn't work (or that `scopes`/nearest-config-discovery was somehow
still broken), when the actual problem was that the fixed code was never loaded at all. See
Footgun 3 below for the general mechanism this is a real instance of.

**The general, still-worth-following convention, independent of whether every tool in a given
pipeline gets this right**: when adding a TEMP local-path override's paired `scopes` block, prefer
an alias NAME that does NOT already exist in the consuming project's own top-level `imports` —
even though a spec-compliant resolver handles the collision correctly, a real dev-server/bundler
bridge sitting on top of it might not thread the referrer it needs to resolve the collision at all
(this exact class of bug — silently reachable only through SOME resolution paths and not
others — is what every footgun in this skill is an instance of). Renaming the nested `scopes` keys
to something collision-free (e.g. `serverUtils/` instead of reusing `utils/`) removes the
dependency on any one tool's referrer-threading correctness entirely, at zero cost — the override
still needs to redirect every bare specifier the linked dependency's own source uses, it just
doesn't have to share the string with something the top-level `imports` already claims for a
different directory. Reaching for a collision-free name in a NEW TEMP override is strictly cheaper
than diagnosing which layer of a multi-tool resolution pipeline doesn't honor `scopes` correctly.

## Footgun 2 — a `scopes` prefix pointed at a linked package's own ROOT directory can silently fail to override that package's OWN top-level `imports`; scope it at `.../src/` instead

A real, confirmed-via-repro footgun distinct from Footgun 1. When pairing a TEMP local-path override
with a `scopes` block meant to pin one of the linked package's OWN bare specifiers back to a real
published range (e.g. the registry-singleton fix pattern `deno-lazy-dependency-pattern` documents,
where a linked package's own further link to a THIRD package would otherwise create a duplicate
module instance), the `scopes` KEY's own prefix matters, and the natural-looking choice — the
linked package's bare checkout directory (`"../datamaster/"`) — does NOT reliably work, even though
it is a valid prefix of every file under it.

**Confirmed via a real, isolated `deno info --json` repro (2026-08-25, `@zanix/admin` linking
`../datamaster`)**: `"scopes": {"../datamaster/": {"@zanix/server": "jsr:@zanix/server@^3.4.0"}}`
did NOT override `@zanix/server` for any file under `../datamaster/` — `deno info --json` still
showed every one of `../datamaster/mod.ts`'s (and its whole `src/` tree's) own `@zanix/server`
imports resolving to the OTHER linked package's local checkout (`../server/mod.ts`, `datamaster`'s
own separate TEMP override), not the pinned real published range — reproducing the exact
duplicate-module-instance risk the scope was written to prevent. Changing ONLY the scope key's
prefix — from `"../datamaster/"` to `"../datamaster/src/"` — fixed it immediately, confirmed by the
same `deno info --json` repro then showing every file correctly resolving `@zanix/server` back to
the pinned real published range. Nothing else about the override or the scope's own value changed.

**Why this matches the ecosystem's own existing precedent, once you look for it**: every
already-working `scopes` pin `deno-lazy-dependency-pattern` documents (`datamaster`'s own
`"../server/src/"` scope) already uses the `.../src/` prefix, never the bare checkout root — this
was previously undocumented as a
REQUIREMENT (it read as if any of the linked package's own file paths would work as a prefix, since
`scopes` matching is a plain string-prefix check per the WHATWG Import Maps spec). The real,
still-unconfirmed mechanism: a bare checkout root prefix likely collides with Deno's own separate
"workspace member" auto-discovery for a directory whose `deno.json(c)` declares a real
`name`+`version` (the same mechanism Footgun 4 below describes) — that auto-discovery may take
priority over a `scopes` entry keyed at the package's OWN root, while a deeper, non-root prefix
doesn't trigger the same collision. Treat this as a confirmed WORKAROUND, not yet a confirmed
root-cause explanation — verify with a real `deno info --json` repro on any NEW TEMP-override +
`scopes` pairing rather than assuming the mechanism generalizes exactly as described here.

**The practical rule**: when writing a `scopes` entry meant to override a linked package's OWN bare
specifier, always key it at that package's real source-tree prefix (`.../src/`, or whatever the
package's own `deno.jsonc` `imports` map shows its internal aliases resolving relative to — check
that file, don't guess), never the bare checkout directory, even though the bare directory is also
a syntactically valid (and, per spec, should-also-work) prefix.

## Footgun 3 — a TEMP override on a package's bare `.` export does NOT cover its other subpaths — each one needs its own entry, and a missed one can masquerade as an unrelated bug

A real, general footgun, confirmed via a real, live `deno run` repro (not `deno info --json` alone)
that chased what looked like a `scopes`/nearest-config-discovery problem for hours before finding
the real cause (`@zanix/space`/`console`/`@zanix/cli`, 2026-08-25): **a real `jsr:pkg@version`
specifier gets automatic subpath expansion — declaring the bare package once covers every subpath a
consumer might import — but a raw relative-path override (`"@zanix/pkg": "../pkg/mod.ts"`) does
NOT.** Each subpath a consumer's own reachable code imports (`@zanix/pkg/dev`, `@zanix/pkg/vite`,
...) needs its OWN separate override entry pointing at that subpath's own local file. Missing one
for a subpath nothing in the CURRENT project's own source imports directly, but that a DEPENDENCY's
own code imports internally (here: `@zanix/cli`'s own `commands/space/dev/command.ts` statically
importing `@zanix/space/dev`), silently falls through to whatever the bare/`.`-only override
didn't touch — the real PUBLISHED version of that one subpath, still running whatever bug the
local checkout's fix was supposed to have already closed.

**Why this is easy to miss and hard to diagnose**: the resulting failure doesn't look like a
missing-override problem at all — it looks exactly like whatever bug the unpublished fix was
supposed to have closed, still happening, which reads as "the fix doesn't work" or "some OTHER
resolution layer (`scopes`, nearest-config-discovery, a dev-server bridge's referrer-threading) is
broken" — because from the consumer's point of view, a real local link to the package IS in place
(the bare `.` override plainly resolves locally, confirmed via `deno info`/a warning citing the
right local path), so "is the package actually linked" looks like a settled question when it
isn't, for the one subpath that matters. **The tell**: a `deno run` warning (or `deno info --json`
`modules` entry) citing an `https://jsr.io/pkg@version/...` URL as the RESOLVED location for the
specific file/symbol involved in the failure — not the bare package's own root, the SPECIFIC
subpath's own entry file — proves that subpath is still resolving to the published version,
regardless of what the bare override says. Check this before trusting any theory about `scopes`,
referrer-threading, or per-package config discovery being the culprit; it is the first, cheapest
thing to rule out. **The fix**: enumerate every subpath of the overridden package that is
REACHABLE from the current run — not just what the current project's own source imports directly,
but what every dependency already resolved to a local path (a CLI tool, a shared library) imports
internally too — and give each one its own override entry to the matching local file, the same way
`console/deno.json`'s own comment already documents doing for `@zanix/space/react` (a deliberate,
explicit separate key, not automatic subpath expansion) — the gap here was doing that
CONSISTENTLY for every subpath actually reached, in every config file in the chain, not just the
ones the investigator happened to think to check.

## Footgun 4 — a raw local-path override anywhere in a config can silently auto-link UNRELATED sibling directories too

A distinct, real Deno mechanism from the three above — this one isn't about the overridden
package's OWN specifiers resolving unexpectedly (that's Footgun 3's "linked package by name" case:
introducing a raw relative-path reference into a directory whose `deno.json(c)` declares a matching
`name`+version makes Deno auto-resolve EVERY specifier for THAT SAME package project-wide, not just
the one that used the raw path). This one is about OTHER, otherwise-untouched sibling directories
getting pulled in. Confirmed via a real, isolated same-config-different-directory repro: the moment
a config (`cli/deno.jsonc`, in the confirmed case) has ANY raw relative-path override into ONE
sibling checkout (e.g. `@zanix/server` → `../server/mod.ts`), Deno silently discovers and
substitutes OTHER sibling directories under the same parent too — any directory whose own
`deno.json(c)` `name`+version happens to satisfy some UNRELATED bare specifier elsewhere in the
graph — with zero explicit configuration asking for it and no warning when it matches.

**Why this is dangerous**: it can substitute a completely unrelated sibling package's own
uncommitted, broken, in-progress work for the real published version, in a config that never
mentioned that package at all — making an unrelated repo's own WIP bug look exactly like a defect
in the config being investigated. Confirmed real case: auditing `cli`'s own dependency overrides
(which only referenced `@zanix/server` and `@zanix/asyncmq`) silently picked up `@zanix/space`'s
own broken, uncommitted `build-client.ts` change instead of the real published `@zanix/space`,
because `../space`'s own `deno.jsonc` declared a `name`+version that happened to satisfy a bare
`@zanix/space` specifier reached elsewhere in the same graph — burning real investigation time on
what looked like a `cli` regression.

**The tell, same discipline as Footgun 3**: before trusting a theory about what a config change
broke, check whether EVERY sibling directory's own `deno.json(c)` is in the exact state you think
it's in (`git status`/`git diff` in each sibling repo, not just the one you changed) — a raw
local-path override anywhere in the chain is reason enough to distrust every sibling's resolved
version until checked directly, not just the one path segment you wrote.

## Checklist before trusting a TEMP local-path override is fully, correctly wired

- [ ] Does the paired `scopes` block (if any) use an alias NAME that doesn't collide with the
      consuming project's own top-level `imports` — or, if it must collide, has the actual
      bundler/dev-server bridge in the pipeline been confirmed (not assumed) to thread a real
      referrer through to the resolver (Footgun 1)?
- [ ] Is every `scopes` key prefix set at the linked package's real source-tree root (`.../src/` or
      equivalent — checked against that package's own `deno.jsonc`, not guessed), never the bare
      checkout directory (Footgun 2)?
- [ ] Has EVERY subpath of the overridden package that's reachable from the current run — not just
      what the current project's own source imports directly, but what any dependency (a CLI tool,
      a shared library) already resolved to a local path imports internally too — gotten its own
      explicit override entry (Footgun 3)?
- [ ] Before trusting a theory about a broken resolver/scope/referrer, checked what a `deno run`
      warning or `deno info --json` `modules` entry says the SPECIFIC failing file/symbol actually
      resolved to (Footgun 1 & 3's shared tell)?
- [ ] Before trusting a theory about what a NEW config change broke, checked `git status`/`git diff`
      in EVERY sibling repo reachable via any raw local-path override in the chain — not just the
      one file you edited (Footgun 4)?

## Out of scope — do not do these

- The over-materialization mechanism a TEMP link is usually deployed to verify a fix for — that's
  `deno-lazy-dependency-pattern`'s job, not this skill's.
- Deciding whether to use a TEMP local-path override at all versus the `links` array — this skill
  assumes that decision is already made; see the Golden Rule above for the one-line reason a raw
  path is usually preferred for this exact workflow.
- Fixing a specific tool's own referrer-threading bug (e.g. `@zanix/space`'s dev-server bridge) —
  that's a real, tool-specific bug fix outside this skill's own scope; this skill documents the
  SYMPTOM and the general mitigation (a collision-free scope name), not every tool's own internals.

---
name: cli-generator-testing
description: Testing architecture for @zanix/cli's generators and scaffolds — the three-tier unit/integration/functional suite mirrored against src/commands/generate/, and how it complements Drift Watch/--verify for validating what new/generate actually produce. Use when adding tests for a new generator, or deciding where a new test belongs.
---

This skill refines `zanix-test-tier-conventions`'s ecosystem-wide default
with `cli`-specific detail — load both, this one doesn't repeat the general
tier definitions. It covers how `cli`'s own test suite is organized — what
proves a generator produces the right deterministic output, and what proves
the output is actually usable. That second question (does generated code stay compiling
against real, currently-published dependency versions) is a distinct concern
covered fully by `cli-dependency-compatibility` (Drift Watch, `--verify`); this
skill covers the ordinary test suite Drift Watch sits alongside. File:line
references point at `~/Documents/Development/ZanixLibraries/cli` — read the real
code there before assuming this summary is still accurate.

## Golden rule (token savings)

- **Run the full suite (`deno test -A`) at most twice per feature: once at the
  very start, to establish a clean baseline, and once at the very end, as the
  final Validation gate.** While iterating, run only the new/changed test
  file(s) directly (`deno test src/@tests/unit/commands/generate/<artifact>/`) —
  a full multi-hundred-test run on every iteration is the single biggest
  avoidable cost in this whole workflow, and it doesn't tell you anything a
  scoped run doesn't while you're still writing the generator.
- **A pre-existing failure gets one differential check, not an investigation.**
  If the baseline run already showed it failing before any of this feature's
  files existed, that's enough to rule it out — name it in the report as
  pre-existing/unrelated and move on. Don't re-run the full suite repeatedly
  trying to double-confirm it, and don't read into the failing test's own
  history looking for a root cause that isn't this feature's job to fix.
  **Confirmed real, not hypothetical**: `cli`'s own `deno check`/full suite can
  be repo-wide broken by a known cross-repo drift (e.g. the
  `SESSION_COOKIE_ATTRIBUTES` promotion gap) that has nothing to do with any
  generator work — when that's the case, the Validation step's "full suite
  green"/"`deno check` clean" requirement is literally unsatisfiable to the
  letter; a documented differential check (run the same command against an
  unrelated sibling generator, confirm the identical failure) is the correct,
  sufficient substitute, not a blocker to escalate or work around by other
  means.
- **A known coverage-tooling artifact gets confirmed once, not re-derived per
  generator.** If line-coverage looks capped below 100% for a reason that isn't
  a real gap (e.g. a builder-chain body every `register<Name>Command` shares),
  check ONE sibling generator's coverage number to confirm it's the same
  artifact there too — don't re-investigate the same tooling quirk from scratch
  for every new generator.
- Report the result once, in the compact form this skill's checklist implies
  (pass/fail per tier, coverage %, one line per pre-existing failure) — not a
  running narrative of every command run along the way.

## Three tiers, mirrored against the source they test

`src/@tests/{unit,integration,functional}/`, configured via `deno.jsonc`'s
`"test": {"include": ["src/@tests/**/*.test.ts", "scripts/**/*.test.ts"]}`:

- **Unit** (`src/@tests/unit/commands/generate/<artifact>/command.test.ts`) —
  one test file per generator, mirroring `src/commands/generate/`'s own
  structure exactly. `rto` additionally gets its own `parser.test.ts`/
  `renderer.test.ts`, matching its parser/renderer split (see
  `cli-artifact-generators`). Adding generator #N means adding its own test file
  at the matching path — the folder structure is the map. **This tier's own
  scope is explicitly generators under `commands/generate/`** — a brand-new,
  non-generator top-level command (a `build`/`prepare`-shaped single-leaf
  command, not a `generate <artifact>`/`new <type>` leaf) does NOT
  automatically belong here just because it's new. Check the closest real
  precedent's actual tier first (see Integration below) rather than
  defaulting a new top-level command to `unit/` by habit.
- **Integration** (`src/@tests/integration/`) — cross-cutting checks that span
  more than one generator or more than one project type: preset equivalence and
  fail-fast-on-unknown-preset across all five project types, tree-shape
  assertions, `docker`/`github` scaffold checks (`zanix prepare`), `build`
  checks. **This is also the real, confirmed precedent for a single-leaf
  top-level command's own tests** (`build.test.ts` calls
  `compileAndObfuscate` directly, doing real file I/O rather than fully
  mocking it) — a new command shaped like `build`/`prepare` (not a
  `generate`/`new` leaf) belongs here by default, not in `unit/`.
- **Functional** (`src/@tests/functional/`) — real CLI subprocess runs producing
  real files on disk (`commands.new.test.ts`, `commands.generate.test.ts`,
  `commands.prepare.test.ts`, a JSR-fetch test for the `library` exception).
  This is the tier that proves the CLI as a whole binary behaves correctly, not
  just its internal functions.

  **Known real gap, don't silently replicate it as a new "no functional test"
  precedent**: as of this writing, none of the 6 frontend artifacts
  (`comet`/`page`/`layout`/`error`/`loading`/`not-found`) has its own
  functional-test entry, even though this checklist requires one — only the
  backend artifacts do. When adding a NEW frontend-family artifact, add the
  functional test anyway (the checklist's own letter, and the shape every
  backend generator already follows) rather than matching the immediate sibling
  family's gap — a gap repeated by every new addition never closes. If a future
  task is specifically about closing this gap for the existing 6, that's real,
  separate, in-scope work, not something to bundle silently into an unrelated
  generator addition.

## The Validation step, concretely

Every generator feature's standing workflow (`cli-artifact-generators`) ends in
a Validation step with four parts, all required before calling a generator
"done":

1. `deno check` the generated **output** against real, currently-published
   dependency versions — not the generator's own source, the _files it writes_.
   This is what Drift Watch/`--verify` automate on an ongoing basis
   (`cli-dependency-compatibility`); doing it once by hand during development is
   still part of finishing the feature.
2. 100% branch/function coverage on the new code.
3. Full test suite green.
4. `deno lint`/`deno fmt --check` clean.

A generator whose snapshot test passes but whose real output has never been
`deno check`ed against a live project is not done — a deterministic-output test
only proves the generator is consistent with itself, never that what it produces
actually compiles.

**`feature-completeness-conventions`'s JSDoc gate (`deno doc --lint` passing)
still applies on paper, but confirm this before treating it as a real blocker**:
as of this writing, `deno doc --lint` fails on every existing generator's own
source in this package (not the generator's _output_ —
`command.ts`/`template.ts` themselves), for reasons outside any single
generator's scope (undocumented interface fields, `Commander` not exported from
`cli.ts` for a JSDoc `@link` to resolve). No task or CI in this repo currently
runs this check. Before spending effort "fixing" this for a new generator
specifically, run the same `deno doc --lint` against one sibling generator's own
file — if the failure is the identical pattern, it's pre-existing and out of
scope (same differential-check discipline as any other pre-existing failure
above), not something the new generator broke or is expected to solve alone.

## Keep a variant axis derived from real source, not hand-duplicated

`handler/command.ts` deliberately exports `HANDLER_TYPES` (its own closed
`--type` enum) specifically so Drift Watch's variant matrix
(`cli-dependency-compatibility`) can import it directly, rather than maintaining
a second, parallel list of handler types that could silently drift from the
generator it's meant to be watching. Apply the same pattern to any future
generator with a closed option set: export the enum, don't let a test or CI
script re-type it by hand. Options with an open-ended shape (`--slot`,
`--field`) have no closed set to derive from — those variants stay curated by
hand, chosen to hit every distinct code path rather than exhaustively.

## Checklist before calling a generator's tests done

- [ ] Does a unit test exist at the path mirroring the generator's own module
      location, not just somewhere in the suite?
- [ ] For a brand-new, non-generator top-level command: does its test
      follow `build.test.ts`'s real precedent (`integration/`, real I/O)
      rather than being placed in `unit/` by default?
- [ ] For a DSL-based generator (parser/renderer split), are both halves tested
      independently, not just the combined output?
- [ ] Does a functional test exercise the real CLI subprocess for at least the
      common case, not only the in-process unit-tested function?
- [ ] Has the generated _output_ actually been `deno check`ed against a real
      project — by hand during development, and covered going forward by Drift
      Watch/`--verify` (see `cli-dependency-compatibility`)?
- [ ] If this generator has a closed option set (like `handler --type`), is it
      exported from the generator's own source for Drift Watch to reuse, rather
      than hand-duplicated in the test/CI script?

---
name: feature-completeness-conventions
description: General engineering gate tying together test coverage, README/docs, and JSDoc audits — every new feature must satisfy all three before being called done, and every change/adjustment to existing functionality must re-verify (not just re-run) that its tests, docs, and JSDoc still match reality. Also covers file-size discipline (when a file's grown large enough to split), and two conditional triggers — running security-review when a change touches a security-relevant surface, and checking an existing benchmark suite when one exists and the change touches a measured path. Use it when adding a new feature, modifying existing behavior, or reviewing a PR/diff for completeness before merge.
---

This skill is the umbrella convention over three sibling audit skills —
`complete-test-coverage`, `docs-readme-audit`, `jsdoc-jsr-audit` — it does not
replace their mechanics, it defines WHEN each one must be applied and what
"done" means for a change. Delegate the actual heavy-lifting (coverage triage,
link/anchor scripts, doc-lint iteration) to those skills; use this one to decide
scope and to avoid the most common failure mode: a change ships with green tests
but stale docs/JSDoc, or a new feature ships with docs but no real test behind
them. For CHANGELOG/version/release mechanics once a change is complete, see
`release-management` — this skill stops at "is the change itself done," not
"how do we ship it."

## Golden rule

- **Classify the change first** (Phase 0), then apply only the matching
  checklist. Don't run a full package-wide audit for a one-line fix — scope the
  review to the blast radius of what actually changed (Phase 3). Full sweeps via
  the sibling skills are periodic/on-demand; this skill is the per-change
  discipline that keeps drift from accumulating between those sweeps.
- **"Tests still pass" is not the bar.** A test can stay green after a behavior
  change simply because it never asserted the part that changed. The bar is:
  does a test exist that would FAIL if the new behavior regressed back to the
  old one? If not, it's not covered, regardless of color.
- **A stale doc/JSDoc that contradicts new behavior is worse than a missing
  one.** Don't defer "update the docs" to a follow-up task — an outdated
  `@throws`, default, or README example is an active bug in what it claims, not
  a nice-to-have.
- **Non-Deno/non-JSR stacks**: `complete-test-coverage` already applies as-is.
  For docs/JSDoc, apply the same PRINCIPLES from
  `docs-readme-audit`/`jsdoc-jsr-audit` (symbol cross-check, no stale claims,
  full exports→docs coverage) against whatever the stack's equivalent is (TSDoc,
  docstrings, Sphinx, godoc, etc.) — the mechanics differ, the bar doesn't.
- **CHANGELOG entries are tracked as you go, written at an explicit trigger —
  see `release-management` for the full discipline** (including why writing
  to `CHANGELOG.md` after each micro-edit is wrong, not just untidy).
- **File-size ceilings (below) are a split candidate, not a blocker.** Don't
  refactor an unrelated over-ceiling file as a side effect of a change that
  happens to touch it — note it and move on, unless splitting it is the
  actual task.

## Phase 0 — Classify the change

Before deciding what to check, decide what kind of change this is:

1. **New feature** — a net-new exported symbol, endpoint, option, or observable
   behavior.
2. **Modification/adjustment** — an existing symbol's signature, default, error
   condition, or return shape changed.
3. **Bugfix** — observable behavior changes to match what was always
   intended/documented.
4. **Pure internal refactor** — no observable behavior change, nothing
   public-facing moved.

Refactors with genuinely no external-facing change skip Phase 1/2 entirely
(there's nothing new to document, and existing tests should already cover the
preserved behavior — if they don't, that's a pre-existing gap, handle it via
`complete-test-coverage` on its own terms). Everything else goes through the
matching gate below.

## Phase 1 — New feature gate (all three required before "done," plus a security check every time)

A PR introducing new functionality is incomplete if any of these is missing —
flag it explicitly, don't let it slide to "a follow-up":

- **Tests**: write real coverage as you write the feature, applying
  `complete-test-coverage`'s Phase 2 triage inline (real gap vs. dead code vs.
  env-dependent) rather than deferring to a later audit pass.
- **Docs**: the new symbol needs a documented home — a full example if it's
  user-facing, a compact reference row if it's low-level plumbing
  (`docs-readme-audit` Phase 2 criteria) — plus a CHANGELOG candidate under
  `Added` (see `release-management` for when to actually write it).
- **JSDoc**: the new exported symbol needs accurate JSDoc that would pass
  `deno doc --lint` (or the stack's equivalent doc-lint/coverage check) — not
  just a one-liner restating the name.
- **Security check (always asked, not always triggered)**: does this new
  functionality touch any surface on the trigger list in "Security" below —
  auth/sessions, a guard's default, input parsing, secrets, cryptography,
  file paths, cookies/CSRF/CSP? A brand-new feature is exactly where a fresh
  vulnerability gets introduced, not just where an old one gets found — don't
  reserve this question for a dedicated security pass that may never get
  scheduled. Run `security-review` if triggered; say "no security surface
  touched" if not — either way, the question gets asked, every time, same as
  Tests/Docs/JSDoc.

## Phase 2 — Modification/adjustment gate (the part most easily skipped)

This is the gate that catches drift: changing something that already shipped
without touching what already describes it.

1. **Tests** — find every existing test exercising the changed function/branch
   (`grep` for the symbol across the test tree). For each:
   - Confirm it still passes AND that it actually asserts the **new** behavior —
     not that it merely survived because it never touched the changed path.
   - Check whether a test still asserts the **old** behavior and now silently
     contradicts the new one (a stale expectation that happens to still pass is
     a landmine, not a pass).
   - Any new branch/parameter/error path the change introduces needs coverage —
     triage it like any other gap per `complete-test-coverage` Phase 2.
2. **Docs** — `grep` README/`docs/` for every mention of the changed symbol
   (signature snippets, documented defaults, described behavior,
   `@throws`-equivalent prose). If the change alters a default, a parameter's
   meaning, an error condition, or a return shape that's documented anywhere,
   update it there — don't leave the old prose next to the new code. Note a
   CHANGELOG candidate (`Changed`/`Fixed`/`Deprecated`) if the change is
   public-facing (see `release-management` for when to write it).
3. **JSDoc** — re-verify, don't just re-run lint. Apply `jsdoc-jsr-audit` Phase
   A's standard to the changed symbol specifically: does
   `@param`/`@returns`/`@throws`/`@default`/execution-order prose still describe
   what's true today? A behavior change that leaves the old JSDoc sentence in
   place is exactly the "doc that lies" failure mode that skill exists to catch
   — it's not automatically caught by doc-lint, which only checks
   presence/reachability, not accuracy.
4. **Security check** — the same trigger question as Phase 1, asked again
   here specifically because a modification is where an existing safe
   default quietly becomes unsafe (a guard relaxed, a sanitizer's edge case
   changed, a cookie/CSRF constraint loosened) without anything about the
   change *looking* like a new feature that would prompt the question on its
   own. Same trigger list, same "run `security-review` or say why not" rule
   as Phase 1.

## Phase 3 — Scoping the review (token discipline)

- Scope test/doc/JSDoc review to the symbols actually touched by the diff —
  `git diff --name-only`/`git diff` for the changed hunks, then `grep` those
  symbol names across tests, README, `docs/`, and the file's own JSDoc. Don't
  re-audit the whole package for a small change.
- Exception: widen the search when the change touches a widely-referenced core
  primitive (a shared type, a default consumed by several decorators/handlers) —
  scope to blast radius, not to "the file that was literally edited."
- If the change is large enough that scoped grepping stops being reliable (e.g.
  a rename touching dozens of call sites), that's the signal to run the full
  sibling skill (`complete-test-coverage`, `docs-readme-audit`, or
  `jsdoc-jsr-audit`) instead of trying to hand-scope it.

## File-size discipline

A fourth axis, orthogonal to Tests/Docs/JSDoc: a file that keeps growing past
a real size is a maintainability signal worth checking at the same point as
the rest of this gate, not a separate audit. Soft ceilings — a prompt to
consider splitting by responsibility, not a build failure:

| Kind | Ceiling | Why |
| --- | --- | --- |
| Production code (`.ts`/`.tsx`, non-test) | ~500 lines | Past this, a file is usually mixing more than one responsibility — split by responsibility (one handler/interactor/provider per concern) before the file becomes its own maintenance burden. |
| Test files (`.test.ts`/`.test.tsx`) | ~1000 lines | Tests are linear enumeration, not branching complexity, so they legitimately run longer — real Zanix test files today already range up to ~1265 lines. The ceiling exists to catch unbounded growth, not to flag every already-large suite; split by scenario/`describe` block once a file stops being one coherent unit. |
| Docs (`.md`) | ~600 lines | Past this, a doc is hard to navigate and usually wants splitting into `docs/*.md` by topic — the pattern most Zanix packages already follow (a lean `README.md` plus focused `docs/` files) rather than one growing monolith. |

**No Deno/JSR tooling enforces this automatically** — `deno lint` has no
built-in max-lines-equivalent rule, and no repo in the ecosystem has a custom
one today. Check with a plain `wc -l` on the files actually touched by the
change (same Phase 3 scoping — don't sweep the whole repo for a small
change); note a file that crosses its ceiling as a candidate for splitting,
don't silently split it as a side effect of unrelated work.

## Security: a trigger check, not a per-change audit

This skill doesn't reimplement source-level security review — that's the
generic `security-review` skill's job, and package-specific footguns already
live in each area's own skill (`datamaster-data-protection`,
`datamaster-storage`, `datamaster-triggers`, `zanix-local-api-implementation`).
What belongs here is the trigger: does this change touch a
security-relevant surface at all? Concretely — auth/session handling,
a guard's default (allow vs. deny), input parsing/sanitization, secrets or
env-var-backed keys, cryptography (encrypt/hash/mask), file-path
construction, interpolating a value into a raw text protocol (a
Memcached/Redis-style key or command containing CRLF or protocol
delimiters), or a cookie/CSRF/CSP mechanism. If yes, run `security-review`
(or the matching package skill's own cautions) before calling the change
done — flagged explicitly, same as an unchecked Tests/Docs/JSDoc box, never
silently assumed fine because the change "looked" small. If no security
surface is touched, say so in one line rather than omitting the row.

## Code language and tone: variable names, comments, everything not covered by JSDoc/docs

`jsdoc-jsr-audit` covers JSDoc comments; `docs-readme-audit` covers
README/`docs/`/CHANGELOG. What's left — identifiers (variables, functions,
classes, types) and inline (non-JSDoc) code comments — follows
`documentation-voice`'s same rule. Apply it to every new/changed line
touched by the diff (Phase 1/2 scope, same as everything else in this
gate) — not a full-repo naming sweep for a small change.

## Performance: only where a real benchmark suite already exists

Most Zanix packages have no benchmark infrastructure, and this skill doesn't
invent one — check `deno.jsonc` for a `bench` task before assuming a package
has this axis at all. Two real ones exist today: `@zanix/server`
(`deno task bench`, plus a baseline-comparison gate — `bench:baseline`
records a reference, `test:perf` re-checks against it) and `@zanix/space`
(`bench:renderer` for `src/@tests/benchmarks/renderer/**/*.bench.ts`,
`bench:space` for a separate manual-spike scenario runner that produces no
pass/fail metric and isn't an acceptance criterion). Where a package has
this infrastructure and the change touches a path an existing `.bench.ts`
scenario measures, run the benchmark (baseline comparison where the package
has one, like `server`'s) before/after and report the delta — not for every
change, only one plausibly affecting a measured hot path. A package with no
benchmark suite gets no performance row at all, not an apology for missing
one. For a periodic/on-demand full sweep across every repo with this
infrastructure, or for turning a raw bench/`test:perf` run into a
digestible report rather than a single spot-check delta, see
`benchmark-sweep` — that agent runs the same tasks named here and does the
report-interpretation work, so this section stays the trigger question, not
the execution/analysis mechanics.

## Phase 4 — Gate before calling it done

```
[ ] Tests     — new/changed behavior has real coverage; no stale assertion of the old behavior remains
[ ] Docs      — the symbol has a documented home; nothing left contradicts the new behavior; CHANGELOG
                candidate noted if public-facing (written at the trigger point — see `release-management`)
[ ] JSDoc     — accurate on every changed/new symbol; doc-lint (or stack equivalent) doesn't regress
[ ] File size — touched/new files checked against the ceilings above; over-ceiling files noted as
                split candidates, not silently ignored or silently split
[ ] Security  — checked against the trigger list above; security-review run if triggered, or
                explicitly noted as "no security surface touched"
[ ] Language  — new/changed identifiers, inline comments, docs, and JSDoc are English, present
                tense, free of session/plan/stage narrative (real `@deprecated` excepted)
[ ] Perf      — checked only if the package has a real benchmark suite AND the change touches a
                measured path; explicitly noted as "no benchmark suite" or "not a measured path"
                otherwise
[ ] Version   — bump considered if the project uses semver and the change is public-facing
                (see `release-management`)
```

Any unchecked box is either fixed before calling the work done, or reported
explicitly with a reason (e.g. "internal-only change, no public doc surface
applies") — never silently skipped.

**A consuming agent's own "Definition of done" must cite this full 8-item
block, not shorthand it to "Tests, Docs, JSDoc."** Confirmed real: a
session-narrative inline comment — narrating a dependency's entire removal
history ("was only ever pulled in by...", "which is gone now...", "audited
every other generator too...") instead of just stating why the current
list is what it is, exactly what `documentation-voice`'s Language rule
forbids — landed in `cli`'s `src/utils/config/dependencies.ts` uncaught,
because the dispatching agent's own "Definition of done" section cited
only a 3-item subset (Tests/Docs/JSDoc) of this checklist. Language was
never actually checked, even though both this skill and
`documentation-voice` already state the rule clearly. Naming "Phase 1
gate" or "Tests/Docs/JSDoc" alone is a real, confirmed way to silently
drop Language/File size/Security/Perf/Version from what an agent actually
applies — cite this full Phase 4 block (or a skill's own equivalent
per-mechanism checklist, if one already subsumes it), or state explicitly
which of its 8 items don't apply this time and why.

## Expected report format (to avoid wasting tokens narrating)

```
Change type: <new feature | modification | bugfix | pure refactor>

Tests:  <new file:line covered> / stale assertions fixed: <file:line, one sentence each>
Docs:   <what was added/updated, with file:line> / CHANGELOG candidate: <noted, or "not public-facing">
JSDoc:  <symbols re-verified and fixed, file:line> / doc-lint: <no regression, confirmed via git stash>
Size:   <files over ceiling, with line count, or "none over ceiling">
Security: <surface touched + security-review outcome, or "no security surface touched">
Language: <violations fixed, file:line, or "clean">
Perf:   <benchmark run + delta, or "no benchmark suite" / "not a measured path">

Out of scope (with reason):
- <file:line> — <why this gate doesn't apply here>
```

For the CHANGELOG/version/commit sequence once this gate passes, see
`release-management`.

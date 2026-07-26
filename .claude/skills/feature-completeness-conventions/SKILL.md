---
name: feature-completeness-conventions
description: General engineering gate tying together test coverage, README/docs, and JSDoc audits — every new feature must satisfy all three before being called done, and every change/adjustment to existing functionality must re-verify (not just re-run) that its tests, docs, and JSDoc still match reality. Use it when adding a new feature, modifying existing behavior, or reviewing a PR/diff for completeness before merge.
---

This skill is the umbrella convention over three sibling audit skills — `complete-test-coverage`,
`docs-readme-audit`, `jsdoc-jsr-audit` — it does not replace their mechanics, it defines WHEN each
one must be applied and what "done" means for a change. Delegate the actual heavy-lifting (coverage
triage, link/anchor scripts, doc-lint iteration) to those skills; use this one to decide scope and
to avoid the most common failure mode: a change ships with green tests but stale docs/JSDoc, or a
new feature ships with docs but no real test behind them.

## Golden rule

- **Classify the change first** (Phase 0), then apply only the matching checklist. Don't run a full
  package-wide audit for a one-line fix — scope the review to the blast radius of what actually
  changed (Phase 3). Full sweeps via the sibling skills are periodic/on-demand; this skill is the
  per-change discipline that keeps drift from accumulating between those sweeps.
- **"Tests still pass" is not the bar.** A test can stay green after a behavior change simply
  because it never asserted the part that changed. The bar is: does a test exist that would FAIL if
  the new behavior regressed back to the old one? If not, it's not covered, regardless of color.
- **A stale doc/JSDoc that contradicts new behavior is worse than a missing one.** Don't defer
  "update the docs" to a follow-up task — an outdated `@throws`, default, or README example is an
  active bug in what it claims, not a nice-to-have.
- **Non-Deno/non-JSR stacks**: `complete-test-coverage` already applies as-is. For docs/JSDoc,
  apply the same PRINCIPLES from `docs-readme-audit`/`jsdoc-jsr-audit` (symbol cross-check, no stale
  claims, full exports→docs coverage) against whatever the stack's equivalent is (TSDoc, docstrings,
  Sphinx, godoc, etc.) — the mechanics differ, the bar doesn't.
- **CHANGELOG entries are tracked as you go, written at an explicit trigger, not on every edit.**
  Don't write to `CHANGELOG.md` after each micro-change — that fragments one logical change into
  noise and risks a premature version bump for work still in flux. See Phase 5 for exactly when to
  write/consolidate it.

## Phase 0 — Classify the change

Before deciding what to check, decide what kind of change this is:

1. **New feature** — a net-new exported symbol, endpoint, option, or observable behavior.
2. **Modification/adjustment** — an existing symbol's signature, default, error condition, or
   return shape changed.
3. **Bugfix** — observable behavior changes to match what was always intended/documented.
4. **Pure internal refactor** — no observable behavior change, nothing public-facing moved.

Refactors with genuinely no external-facing change skip Phase 1/2 entirely (there's nothing new to
document, and existing tests should already cover the preserved behavior — if they don't, that's a
pre-existing gap, handle it via `complete-test-coverage` on its own terms). Everything else goes
through the matching gate below.

## Phase 1 — New feature gate (all three required before "done")

A PR introducing new functionality is incomplete if any of these is missing — flag it explicitly,
don't let it slide to "a follow-up":

- **Tests**: write real coverage as you write the feature, applying `complete-test-coverage`'s Phase
  2 triage inline (real gap vs. dead code vs. env-dependent) rather than deferring to a later audit
  pass.
- **Docs**: the new symbol needs a documented home — a full example if it's user-facing, a compact
  reference row if it's low-level plumbing (`docs-readme-audit` Phase 2 criteria) — plus a
  CHANGELOG candidate under `Added` (see Phase 5 for when to actually write it).
- **JSDoc**: the new exported symbol needs accurate JSDoc that would pass `deno doc --lint` (or the
  stack's equivalent doc-lint/coverage check) — not just a one-liner restating the name.

## Phase 2 — Modification/adjustment gate (the part most easily skipped)

This is the gate that catches drift: changing something that already shipped without touching what
already describes it.

1. **Tests** — find every existing test exercising the changed function/branch (`grep` for the
   symbol across the test tree). For each:
   - Confirm it still passes AND that it actually asserts the **new** behavior — not that it merely
     survived because it never touched the changed path.
   - Check whether a test still asserts the **old** behavior and now silently contradicts the new
     one (a stale expectation that happens to still pass is a landmine, not a pass).
   - Any new branch/parameter/error path the change introduces needs coverage — triage it like any
     other gap per `complete-test-coverage` Phase 2.
2. **Docs** — `grep` README/`docs/` for every mention of the changed symbol (signature snippets,
   documented defaults, described behavior, `@throws`-equivalent prose). If the change alters a
   default, a parameter's meaning, an error condition, or a return shape that's documented anywhere,
   update it there — don't leave the old prose next to the new code. Note a CHANGELOG candidate
   (`Changed`/`Fixed`/`Deprecated`) if the change is public-facing (see Phase 5 for when to write it).
3. **JSDoc** — re-verify, don't just re-run lint. Apply `jsdoc-jsr-audit` Phase A's standard to the
   changed symbol specifically: does `@param`/`@returns`/`@throws`/`@default`/execution-order prose
   still describe what's true today? A behavior change that leaves the old JSDoc sentence in place
   is exactly the "doc that lies" failure mode that skill exists to catch — it's not automatically
   caught by doc-lint, which only checks presence/reachability, not accuracy.

## Phase 3 — Scoping the review (token discipline)

- Scope test/doc/JSDoc review to the symbols actually touched by the diff — `git diff
  --name-only`/`git diff` for the changed hunks, then `grep` those symbol names across tests,
  README, `docs/`, and the file's own JSDoc. Don't re-audit the whole package for a small change.
- Exception: widen the search when the change touches a widely-referenced core primitive (a shared
  type, a default consumed by several decorators/handlers) — scope to blast radius, not to "the file
  that was literally edited."
- If the change is large enough that scoped grepping stops being reliable (e.g. a rename touching
  dozens of call sites), that's the signal to run the full sibling skill (`complete-test-coverage`,
  `docs-readme-audit`, or `jsdoc-jsr-audit`) instead of trying to hand-scope it.

## Phase 4 — Gate before calling it done

```
[ ] Tests    — new/changed behavior has real coverage; no stale assertion of the old behavior remains
[ ] Docs     — the symbol has a documented home; nothing left contradicts the new behavior; CHANGELOG
               candidate noted if public-facing (written at the trigger point — see Phase 5)
[ ] JSDoc    — accurate on every changed/new symbol; doc-lint (or stack equivalent) doesn't regress
[ ] Version  — bump considered if the project uses semver and the change is public-facing (Phase 5)
```

Any unchecked box is either fixed before calling the work done, or reported explicitly with a reason
(e.g. "internal-only change, no public doc surface applies") — never silently skipped.

## Phase 5 — CHANGELOG: when to actually write it

- **Track, don't write, as you go.** Keep a running mental (or scratch) note of every candidate
  entry (`Added`/`Changed`/`Fixed`/`Deprecated`) as changes accumulate through the session. Writing
  to `CHANGELOG.md` after every micro-edit fragments one logical change into noise and risks a
  version bump for work that's still in flux.
- **Write/consolidate it only on an explicit trigger from the user** — they ask to commit
  ("commit," "commitear," "subir al repo"), mention "pre-commit," ask to bump the version, ask to
  "register the changelog," say the feature/change is finished, or ask to **publish**/**deploy**
  (see Phase 6 — those two kick off the full commit/push/tag sequence, with this CHANGELOG/version
  step as their first move). Those are the signals to act on. Don't infer "end of session" as a
  trigger — a session has no reliable boundary you can detect — and don't decide on your own that
  "this looks done"; wait for the cue.
- **Consolidate on write**: everything accumulated since the last published entry goes into ONE
  version bump, grouped by category per `docs-readme-audit` Phase 3 — not one entry per
  micro-change, even if several of Phase 1/2's gates fired independently during the session.
- **Suggest the bump, don't just apply one.** Classify it by semver against the whole batch being
  consolidated, and say which one and why in one sentence before applying it:
  - **major** — any removed/renamed public symbol, a required parameter added, or a default/behavior
    change that breaks an existing caller's reasonable expectation.
  - **minor** — a backward-compatible new export/option/feature (Phase 1 territory) with nothing
    breaking in the same batch.
  - **patch** — bugfixes, docs-only changes, or internal changes with no public API shift.
  If the batch mixes categories (e.g. a new feature plus a breaking fix), the highest one wins —
  one minor-looking addition doesn't downgrade a breaking change elsewhere in the same bump.
- **Update the real version, not just the CHANGELOG heading.** If the project is a Deno/JSR package
  (`deno.json(c)` with `"exports"`), bump the `version` field there to match — same as
  `docs-readme-audit` Phase 3.4. The CHANGELOG's `[X.Y.Z]` and `deno.json(c)`'s `version` must never
  disagree; if they already do when you start, flag it before adding a new entry on top of the
  mismatch.
- **Scope limit**: this only covers work committed through Claude's own session. If commits happen
  outside it (a teammate, a separate terminal, another tool), no prompt-level instruction can catch
  that — closing that gap needs a real `pre-commit` git hook that checks `CHANGELOG.md` was touched
  when relevant source changed (see the `update-config` skill for wiring hooks). That's a
  tooling/process concern, not something this skill can enforce by itself.

## Phase 6 — Publish/commit trigger: branch, changelog/version, commit, push, tag

Broader trigger set than Phase 5's: **"publish," "deploy," "commit," "commitear," "subir al repo,"
"push,"** or equivalent phrasing — these mean "ship this," and kick off the full sequence below,
following GitFlow conventions.

1. **Suggest the right branch before committing**, based on the Phase 0 classification:
   - **New feature** → suggest `feature/<slug>` (branched from `develop` if the project keeps one).
   - **Bugfix** → suggest `fix/<slug>` for a routine fix, or `hotfix/<slug>` (branched from
     `main`/`master`) if it's patching something already in production.
   - **Modification/adjustment** → whichever of the two above matches the real intent — adds
     capability = feature, corrects behavior = fix.
   - **Pure refactor** → no branch suggestion needed beyond whatever it's already scoped to.
   Suggest it once; if the user is intentionally committing directly to `develop`/a long-lived
   branch, don't insist — that's their call, not a gate to block on.
2. **Close out Phase 5 first**: write/consolidate the CHANGELOG entry and bump the version (with
   the semver suggestion from Phase 5) if the current batch hasn't gone through it yet — don't
   commit with the entry still sitting as an untracked candidate.
3. **Draft a summarized commit message** from the consolidated CHANGELOG entry and the real diff
   (not a generic "update files"), and get it confirmed by the user before running `git commit` —
   a trigger word authorizes the commit action itself, not an arbitrary message on your behalf.
4. Run `git commit` with the confirmed message, then `git push`.
5. **Tag only on a real release branch, per GitFlow** — check the current branch
   (`git branch --show-current`):
   - **`main`/`master`**: this is a release. Show an explicit warning that the tag/push is
     happening directly on the mainline branch, then `git tag v<X.Y.Z>` (matching the version just
     bumped) and `git push --tags`.
   - **Any other branch** (`develop`, `feature/*`, `fix/*`, `hotfix/*`, `release/*` before it merges
     to `main`/`master`, etc.): do NOT create or push a tag — GitFlow tags only mark a mainline
     release, not work in progress on any other branch.
6. Standard git safety still applies on top of this sequence: never force-push, never skip hooks
   (`--no-verify`), never amend a commit that already happened — a failed pre-commit hook gets
   fixed and re-committed, not bypassed.

## Expected report format (to avoid wasting tokens narrating)

```
Change type: <new feature | modification | bugfix | pure refactor>

Tests:  <new file:line covered> / stale assertions fixed: <file:line, one sentence each>
Docs:   <what was added/updated, with file:line> / CHANGELOG: <candidate noted, entry written (trigger:
        <commit|version bump|user request>), or "not public-facing">
JSDoc:  <symbols re-verified and fixed, file:line> / doc-lint: <no regression, confirmed via git stash>

Out of scope (with reason):
- <file:line> — <why this gate doesn't apply here>
```

If Phase 6 ran (publish/commit trigger), append:

```
Branch:  suggested <feature|fix|hotfix>/<slug> (or: committing directly to <branch>, as requested)
Version: <X.Y.Z> (<major|minor|patch> — <one-sentence reason>), deno.json(c) updated
Commit:  "<confirmed message>" — <sha>
Push:    origin/<branch> updated
Tag:     v<X.Y.Z> created + pushed (main/master release) | skipped (not on main/master)
```

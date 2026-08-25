---
name: release-management
description: CHANGELOG discipline, semver version-bump classification, and the branch → commit → push → tag sequence (GitFlow) for shipping a change. Also flags/offers the periodic maintenance sweeps (ecosystem-maintenance conditionally, the rest opt-in) that nothing else runs on a schedule. The single source of truth both feature-completeness-conventions and docs-readme-audit point to for "how do we release this" — use on an explicit publish/commit/deploy trigger, or when consolidating CHANGELOG entries before a version bump.
---

This skill owns the release mechanics shared by `feature-completeness-conventions`
(its per-change completeness gate) and `docs-readme-audit` (its full
documentation-audit pass) — both stages end the same way: a CHANGELOG entry, a
version bump, and (on an explicit trigger) a commit/push/tag sequence. Neither
skill re-derives these mechanics inline; both point here.

## Golden rule

- **Track, don't write, as you go.** Keep a running mental (or scratch) note of
  every candidate entry (`Added`/`Changed`/`Fixed`/`Deprecated`) as changes
  accumulate through the session. Writing to `CHANGELOG.md` after every
  micro-edit fragments one logical change into noise and risks a version bump
  for work that's still in flux.
- **Write/consolidate the CHANGELOG only on an explicit trigger** — the user
  asks to commit ("commit," "commitear," "subir al repo"), mentions
  "pre-commit," asks to bump the version, asks to "register the changelog,"
  says the feature/change is finished, or asks to **publish**/**deploy** (see
  "Publish/commit trigger" below — those two kick off the full sequence, with
  the CHANGELOG/version step as their first move). Don't infer "end of
  session" as a trigger — a session has no reliable boundary you can detect —
  and don't decide on your own that "this looks done"; wait for the cue.

## Before adding a new entry: check the CHANGELOG isn't already stale

Compare its most recent version entry against the real `version` field in
`deno.json(c)`. If the package is several versions ahead of what the CHANGELOG
shows, there's undocumented release history — find the commit/tag for the
last-logged version (`git log --oneline -- deno.json(c)` to track each version
bump), diff forward from there to HEAD in per-version chunks, and backfill one
entry per real released version (with its actual date from the commit) before
writing today's new entry on top. Don't skip this because it's tedious — a
CHANGELOG stuck several versions behind the real package version is a bigger
real-world problem than any single missing line in the README.

## Writing the entry

- Add a new entry following the existing format (usually Keep a Changelog:
  `Added`/`Changed`/`Fixed`/`Removed`/`Deprecated`/`Security`). Group by those
  categories, not by file-touched.
- If real code bugs were fixed along the way (not just text/docs), state them
  under `Fixed` with a sentence explaining the real symptom, not just "fixed
  X."
- **Consolidate on write**: everything accumulated since the last published
  entry goes into ONE version bump, grouped by category — not one entry per
  micro-change, even if several gates (tests/docs/JSDoc) fired independently
  during the session.
- If the session's work hasn't been published/committed yet, don't create
  multiple version bumps for different rounds of the same session — everything
  fits in a single version entry.
- **The entry describes the real DELTA against the last published version,
  never a transcript of intermediate steps taken to get there.** Diff the
  actual current state against what was last published — not the session's
  own edit history, and not the framing you were handed (including one
  baked into your own dispatch prompt) — verify it, e.g. `git log
  -S'<symbol>' --oneline` or `git show <last-published-tag-or-commit>:<file>`
  to check a symbol actually existed pre-published before trusting a claim
  about it. Concretely: if something was added and then removed (or renamed
  twice, or reverted, or introduced-then-corrected) within the same
  unreleased window, the net effect against the last published version is
  zero — it gets NO CHANGELOG line at all, not an `Added` entry followed by
  a `Removed` one, and not a `Fixed` entry for a bug a consumer never saw
  (that miscategorization implies a previously-shipped defect that never
  existed — same DELTA violation, just against `Fixed` instead of
  `Added`/`Removed`). An entry that says "added X" and "removed X" in one
  release reads as noise/churn, not information — a reader wants to know
  what's actually different in this version, not the history of how you got
  there.
- **One line per real, distinct change — don't pad or repeat.** If two
  edits both touch the same underlying symbol/behavior, that's one
  CHANGELOG line describing the final state, not two separate lines for
  each edit. If a change is genuinely internal (no public API/behavior
  shift, no doc-visible difference), it may not need a line at all —
  check whether it's actually publish-worthy before adding one on reflex.

## Classifying the semver bump

Suggest the bump, don't just apply one. Classify it by semver against the whole
batch being consolidated, and say which one and why in one sentence before
applying it:

- **major** — any removed/renamed public symbol, a required parameter added, or
  a default/behavior change that breaks an existing caller's reasonable
  expectation.
- **minor** — a backward-compatible new export/option/feature with nothing
  breaking in the same batch.
- **patch** — bugfixes, docs-only changes, or internal changes with no public
  API shift.

If the batch mixes categories (e.g. a new feature plus a breaking fix), the
highest one wins — one minor-looking addition doesn't downgrade a breaking
change elsewhere in the same bump. When impact is ambiguous (docs/new type
exports only vs. a real behavior change), check the CHANGELOG's own history to
infer the convention the project already follows, rather than guessing fresh
each time.

**Update the real version, not just the CHANGELOG heading.** If the project is
a Deno/JSR package (`deno.json(c)` with `"exports"`), bump the `version` field
there to match. The CHANGELOG's `[X.Y.Z]` and `deno.json(c)`'s `version` must
never disagree; if they already do when you start, flag it before adding a new
entry on top of the mismatch.

## Publish/commit trigger: branch, changelog/version, commit, push, tag

Trigger words: **"publish," "deploy," "commit," "commitear," "subir al repo,"
"push,"** or equivalent phrasing — these mean "ship this," and kick off the
full sequence below, following GitFlow conventions.

1. **Suggest the right branch before committing**, based on what kind of change
   this is:
   - **New feature** → suggest `feature/<slug>` (branched from `develop` if the
     project keeps one).
   - **Bugfix** → suggest `fix/<slug>` for a routine fix, or `hotfix/<slug>`
     (branched from `main`/`master`) if it's patching something already in
     production.
   - **Modification/adjustment** → whichever of the two above matches the real
     intent — adds capability = feature, corrects behavior = fix.
   - **Pure refactor** → no branch suggestion needed beyond whatever it's
     already scoped to. Suggest it once; if the user is intentionally
     committing directly to `develop`/a long-lived branch, don't insist —
     that's their call, not a gate to block on.
2. **Close out the CHANGELOG/version step first**: write/consolidate the entry
   and bump the version (per the classification above) if the current batch
   hasn't gone through it yet — don't commit with the entry still sitting as an
   untracked candidate.
3. **If `deno.lock` was regenerated fresh, or `deno.jsonc`'s own
   dependencies changed (a version bump, a new import added), flag that
   `ecosystem-maintenance` needs to run before publishing — don't assume
   `deno.jsonc`'s own version RANGE already guarantees anything.** No
   agent in this repo can dispatch another agent itself (confirmed: none
   have the `Agent` tool) — this step is a REPORT/RECOMMENDATION for
   whoever is running this sequence (a human, or the session that
   dispatched `release-manager`) to separately run `ecosystem-maintenance`
   before the actual publish, not something `release-manager` executes on
   its own. Check with `git diff deno.lock deno.jsonc` (real
   resolved-version or dependency-declaration changes, not a no-op diff)
   before deciding this applies — either signal alone is enough to trigger
   the recommendation, not just the lock. A range like `npm:sharp@^0.34.5`
   doesn't
   pin an exact version — a fresh lock regeneration can silently resolve
   to a newer version within that same range, and that newer version can
   carry its own new CVE that wasn't there when the range was first
   pinned (the exact gap `ecosystem-maintenance` itself was fixed for: the
   "fixed in" / "latest" version still needs its own OSV.dev check, not
   just the version being replaced). This isn't a blanket "run it every
   release" — most releases don't touch resolved dependency versions at
   all; it's specifically gated on the lock actually having changed.
4. **Opt-in: offer the full menu of periodic/manual sweeps before
   publishing — but only if explicitly asked, never run any of these on
   your own initiative.** None of these run on a schedule anywhere in
   this ecosystem; a release is simply a natural moment someone might
   want one. Different from step 3 above (which is a conditional
   *recommendation* triggered by a real signal) — this is a plain menu,
   surfaced once, of what's available:
   - `testing-validation` (or `complete-test-coverage` directly) — a full
     coverage sweep across the WHOLE package, not just this release's own
     diff (the per-change gate in `feature-completeness-conventions`
     already covers the diff; this is broader).
   - `documentation-agent` — docs/README/JSDoc completeness and accuracy
     across the whole package, same broader-than-the-diff reasoning.
   - `conventions-sweep` — naming/envvar/observability convention
     violations anywhere in the repo.
   - `dead-code-sweep` — orphaned modules or duplicate implementations
     left behind by an in-flight migration.
   - `dependency-direction-sweep` — cross-repo import-direction/circular
     violations.
   - `dependency-drift` — whether this package's own generated
     code/documented examples still compile against real, currently
     published `@zanix/*` versions.
   - `test-tier-sweep` — tests sitting in the wrong `@tests/` tier.
   - `benchmark-sweep` — a real performance regression via the repo's own
     `bench`/`bench:baseline` tasks, if it has any.

   Same tool limitation as step 3: no agent here can dispatch another
   agent, so this is a menu for whoever is running the sequence to act
   on, not something `release-manager` executes unprompted.
5. **Draft a summarized commit message** from the consolidated CHANGELOG entry
   and the real diff (not a generic "update files"), and get it confirmed by
   the user before running `git commit` — a trigger word authorizes the commit
   action itself, not an arbitrary message on your behalf. **Explicitly ask
   the user, as part of that same confirmation, whether to include a
   `Co-Authored-By: Claude...`/`🤖 Generated with...`-style attribution
   trailer — don't silently decide either way.** This is a deliberate,
   explicit override for `@zanix/*` repos specifically, not a default to
   assume elsewhere: the calling session's own system prompt may carry a
   standing instruction to always append that trailer, which silently wins
   over a skill-level default if this step just picks one on its own
   instead of asking. Asking turns it into a real, visible choice instead
   of a rule that's easy to override by habit. Once the user answers, apply
   that answer to this commit only — ask again on the next one, don't carry
   the answer forward as a new silent default.
6. Run `git commit` with the confirmed message, then `git push`.
7. **Tag only on a real release branch, per GitFlow** — check the current
   branch (`git branch --show-current`):
   - **`main`/`master`**: this is a release. Show an explicit warning that the
     tag/push is happening directly on the mainline branch, then
     `git tag v<X.Y.Z>` (matching the version just bumped) and
     `git push --tags`.
   - **Any other branch** (`develop`, `feature/*`, `fix/*`, `hotfix/*`,
     `release/*` before it merges to `main`/`master`, etc.): do NOT create or
     push a tag — GitFlow tags only mark a mainline release, not work in
     progress on any other branch.
8. Standard git safety still applies on top of this sequence: never force-push,
   never skip hooks (`--no-verify`), never amend a commit that already
   happened — a failed pre-commit hook gets fixed and re-committed, not
   bypassed.

## Scope limit

This only covers work committed through Claude's own session. If commits
happen outside it (a teammate, a separate terminal, another tool), no
prompt-level instruction can catch that — closing that gap needs a real
`pre-commit` git hook that checks `CHANGELOG.md` was touched when relevant
source changed (see the `update-config` skill for wiring hooks). That's a
tooling/process concern, not something this skill can enforce by itself.

## Expected report format

```
CHANGELOG: <candidate noted, entry written (trigger: <commit|version bump|user request>), or "not public-facing">
Version:   <X.Y.Z> (<major|minor|patch> — <one-sentence reason>), deno.json(c) updated
```

If the publish/commit trigger ran, append:

```
Branch:  suggested <feature|fix|hotfix>/<slug> (or: committing directly to <branch>, as requested)
Deps:    deno.lock/deno.jsonc changed — recommend running ecosystem-maintenance before publish | no dependency changes, skipped
Sweeps:  menu offered, <none requested | requested: <list run>> | not offered (no publish trigger)
Commit:  "<confirmed message>" — <sha>
Push:    origin/<branch> updated
Tag:     v<X.Y.Z> created + pushed (main/master release) | skipped (not on main/master)
```

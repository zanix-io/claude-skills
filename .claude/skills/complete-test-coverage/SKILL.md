---
name: complete-test-coverage
description: Audits and completes a project's test coverage thoroughly but token-efficiently — detects invisible files, branches hidden inside "covered" lines, dead code that's actually a bug, and classifies every gap before writing anything. Use it when asked to improve/complete test coverage in any stack (Deno, Node, Python, Go, etc.).
---

Use this as the initial prompt in any project with an existing test suite. Replace
`<runtime>` with the real test runner (`deno test`, `npm test`/`jest`, `pytest`, `go test`, etc.)
and `<coverage-tool>` with its coverage equivalent.

---

## Goal

Complete the project's test coverage **thoroughly but token-efficiently**, aiming for the best
realistically achievable coverage score: find ALL real gaps (including the invisible ones and the
ones hiding inside lines that already look "covered"), discard the ones that aren't real gaps, and
close only the ones that add real signal — without dumping full coverage tables into the chat at
every step or re-reading files unnecessarily.

**"Best possible score" doesn't mean 100% at any cost.** It means: no real gap is left unclosed out
of laziness or because a misleading report was trusted, but dead code, unreachable defensive code,
and code that requires live infrastructure with no injection point still stay out of scope (Phase
2) — chasing those anyway would be exactly the kind of token waste this audit exists to prevent.
The difference versus a shallow pass is that here you don't declare "done" until the number that's
left unclosed is explicitly justified line by line, not just rounded off.

## Golden rule (token savings)

- Analyze in bulk, not interactively. Run the full suite with coverage ONCE, extract the detail to
  a temp file, and do all the triage against that file with `grep`/`sed`/short scripts — don't
  re-run the full suite after every individual test. Run individual files only to iterate quickly
  on something specific, and reserve the full run for: (a) the initial diagnosis, (b) checkpoints
  every ~5-8 new files, (c) the final verification.
- Don't paste full coverage tables into the chat. Summarize: "N files at 100%, these M remain for
  this specific reason." Give detail only if the user asks for it.
- Don't repeat explanations already given. If it's already been established that "the branch % in
  the summary table can be an artifact, the source of truth is the raw branch data (Phase 0.4), not
  the per-line `--detailed` report nor the aggregated summary," don't re-explain it every time —
  just apply it.
- Don't trust that a test "passing" means it closed the coverage gap you were after. If the test
  uses a helper that rebuilds/clones the function under test (see Phase 3), it can pass perfectly
  and not move a single bit in the original file's report. Verify the coverage data after writing
  the test, not just the test result.
- Before writing a test, classify the gap (see Phase 2). The "not real" ones don't get written —
  report them in one line and move on. Chasing 100% on dead code is the single biggest waste of
  tokens possible in this task.
- **Always `rm -rf coverage` (or equivalent) before a full run, and never run two test-runner
  invocations concurrently against the same project.** Coverage profile directories accumulate one
  file per isolate/worker across every run that wrote to them — running the full suite more than
  once without clearing it first silently merges old and new profiles into one contaminated report,
  and any percentage read from it afterward cannot be trusted (a file can show 100% from stale data
  while the current source no longer even exercises that path). Separately, if the project under
  test touches shared external state (a local SQLite/KV file, a message broker, a shared port) — two
  concurrent test-runner processes can deadlock or hang on that shared state (a lock file never
  released by a killed prior run, an orphaned message sitting in a queue). If a run hangs where it
  never did before, suspect leftover state from an earlier interrupted run before suspecting the
  code: check for and clean up stray lock files and any test-created records in shared
  infrastructure, then retry with a single, sequential, freshly-cleared run.

## Phase 0 — Starting point and blind-spot detection

1. Run the full suite with coverage:
   `rm -rf coverage && <runtime> --coverage=coverage` (or the stack's equivalent).
2. **Detect invisible files.** A coverage report only lists files that SOME test loaded. If nothing
   imports a file, it doesn't appear — not even as 0%. Compare the full source tree against the
   union of files that appear in the raw coverage data:
   ```
   find src -name "*.ext" -not -path "*/tests/*" | sort > /tmp/all_src.txt
   # extract unique urls/paths from the raw coverage data (json/lcov/etc.)
   comm -23 /tmp/all_src.txt /tmp/covered_src.txt
   ```
   Exclude type-only/interface files with no executable code (they don't apply). Any real file left
   in the diff is a **total gap** — treat it with maximum priority, these are the cheapest to find
   and the ones that hide the most.
3. Before trusting the "branch" % in the summary table: if the same module gets loaded across many
   isolated test files (shared containers/singletons), that column may not reflect the real union
   of covered branches. Verify against the detailed per-line report (`<coverage-tool> --detailed` or
   equivalent) before writing a test to "close" a branch % — the line might already be at 100% and
   the number might just be aggregation noise across isolated module instances.
4. **The "per-line" report is not enough for branches — it's necessary but not sufficient.** A file
   can show 100% line coverage and still have branches at 0%, because a "covered" line can contain
   a sub-branch that never executed: ternaries (`a ? b : c`), `||`/`??` fallbacks, optional chaining
   (`?.`), default parameter values, and branch selection inside a factory
   (`if (options.each) {...} else {...}`) when only one of the two options was ever tested. Before
   declaring a file "closed," extract the raw branch data (not the aggregated summary) and confirm
   it against the source:
   ```
   # lcov: each BRDA:<line>,<block>,<branch>,<hits> line — hits=0 is the uncovered branch
   awk '/^SF:.*path\/to\/file\.ts/{f=1} f&&/^BRDA:/{print} f&&/^end_of_record/{f=0}' coverage/lcov.info
   ```
   (for other coverage formats, look for the equivalent — istanbul/json exposes `branchMap`+`b` per
   statement, go has `-covermode=count` per block, etc.)
   If a branch is still at 0 despite the line being "green," treat it like any other gap: classify
   it in Phase 2 before deciding whether to close it or discard it.

## Phase 1 — Prioritization

Order the gaps found by impact, not by file:
1. Completely invisible files (Phase 0.2) — top priority.
2. Caches/guards that never trigger (Phase 2, "dead code that's actually a bug") — the single most
   valuable finding possible in this audit, look into it before anything else as soon as you spot
   it.
3. Real uncovered business logic (calculations, validations, error branches with distinct
   behavior).
4. Registration/DI/decorators: "invalid class" branch, default options, guard/middleware
   short-circuits, option-driven branch selection inside a factory (even if it's "hidden" inside a
   line at 100% line%, see Phase 0.4).
5. One-line fallbacks (`||`, `??`, nested ternaries, default callbacks) in real code.
6. Everything else (trivial branches, infrastructure catch-and-ignore).

## Phase 2 — Triage per gap (classify BEFORE writing anything)

For each uncovered line/branch, decide in one sentence:

- **Real, isolated gap** → write a unit test, without touching shared fixtures.
- **Dead code by design** (the check can never be true given how it's called today, e.g. a check
  against a freshly-generated random ID, or a hardcoded constant where a config used to be) → do
  NOT write a test to force it. Report it as a finding and ask whether it should be removed or
  refactored to become reachable (sometimes the right fix is exposing an optional parameter that
  gives the check real meaning, not just deleting it — consider both).
- **"Dead" code that's actually a BUG** — before filing a branch away as dead code, ask yourself:
  *should this branch trigger under normal use and it doesn't because of an implementation error?*
  Typical signal: a cache/memoization or guard (`if (cache?.key === x) return cache`) that never
  matches even when called twice with the same arguments — check whether the value is being
  compared against the wrong variable, or whether the assignment that should populate the cache
  before the `return` is missing. If you find this, it's a more severe finding than a test gap:
  report it separately and ask before touching production code (the fix is usually one line, but
  it's a real behavior change, not just a test change).
- **Depends on real environment** (reads a real config file, uses a module-level cached value) →
  before discarding it, check whether:
  - the function memoizes on first call → you can mock the low-level primitive (file read, fetch,
    etc.) BEFORE that first call, in a dedicated isolated test file for just that branch.
  - it accepts an optional override parameter that the wrapping code doesn't expose — in that case
    it IS a real gap, it just needs to go through the right layer.
  - the fallback you want to force (e.g. "use the default path/cwd") would write to or read from a
    real project location if the real environment happens to match that default path by accident →
    never assume that's implicitly safe. Explicitly redirect ANY write/location parameter to a temp
    folder and use the environment stub (`cwd`, network primitive, etc.) only to decide the branch,
    not for the side effect. Verify at the end with `git status`/diff that nothing outside the temp
    folder was touched.
- **Requires live infrastructure** (real socket, real server, network connection) → don't force it
  with fragile mocks. It's acceptable to leave it without unit coverage if a real integration/e2e
  test already exercises it, even partially. Prioritize reusing that real infrastructure (calling
  the public wrapper twice with a minimal options variation) over building a new mock — it's
  cheaper and more reliable. Never force an external command/service to fail by manipulating global
  process environment variables (`PATH`, etc.): it's not test-isolated and the risk of breaking
  other tests or the process itself outweighs the value of the branch.
  - **Before filing anything under this bucket, check whether it's actually a plain callable in
    disguise.** Message/event handlers (subscribers, consumers, `onmessage`/`onerror` callbacks,
    factory-produced workers) are frequently just methods on a class — instantiate the class
    directly and call the method with a manufactured payload/context, entirely bypassing the
    transport (queue, socket, HTTP) that invokes it in production. "This runs when a real message
    arrives" is not the same claim as "this needs a real message to run" — the second is usually
    false. Only fall back to this bucket once you've confirmed the target genuinely can't be
    invoked without the live transport (e.g. it's private/unreachable, or the transport does
    unskippable work the method depends on).
- **Defensive error handling that should never trigger** (try/catch wrapping a full bootstrap,
  ignore-and-log) → not worth forcing the catch; forcing it usually requires deliberately breaking
  the happy path of everything it wraps.
- **Coverage unit hidden inside a "covered" line** (see Phase 0.4): a parameter with a default
  callback (`function f(cb = (x) => x)`), a factory branch selected by options (`each`, `optional`,
  etc.), a `||`/`??`/`?.` fallback — these ARE real gaps almost always (they're not "trivial" just
  because they're short) and tend to be the cheapest to close once identified. Classify them like
  any other before deciding, but default to treating them as closeable.

## Phase 3 — Test patterns (learned, apply to most stacks)

- **Isolation between test files**: if the runtime isolates each test file (process/worker/fresh
  module per file — verify this once for the stack at hand), you can freely mock globals (`fetch`,
  clock, FS) in a dedicated file without fear of leaking the mock into other files. Document the
  finding if it's not obvious, because it changes the whole mocking approach. Quick, cheap
  verification (two throwaway files, run them together and delete):
  ```
  # a.test.ts: sets a global. b.test.ts: reads that same global.
  # If b sees it as undefined, the runtime isolates per file — mock with confidence.
  ```
- **Test helpers that rebuild the function under test do NOT count toward the original file's
  coverage.** Any utility that takes a function and generates a new one from its source code
  (`new Function(...)`, `eval`, stringify + identifier replacement — the typical hand-rolled
  "mockWrap"/"rewire" helper pattern) creates a function object distinct from the one the coverage
  instrumenter registered. The test can pass perfectly against that clone while the original file's
  line stays red. Detect it like this: if you wrote a test using that kind of helper and the
  line/branch you were trying to close is still at 0 after running it, switch technique — a real
  stub of the low-level primitive (`stub(Deno, 'cwd', ...)`, `stub(globalThis, 'fetch', ...)`,
  etc.) or a direct call to the real function with the arguments that force the branch, never the
  helper that clones code.
- **Factories/decorators run branch selection when invoked, not when used later.** A pattern like
  `function Decorator(options) { if (options.each) {...} else {...}; return
  defineBehavior(...) }` runs its `if/else` the moment `Decorator(...)` is called — typically when
  declaring the class/object that uses it — not when the decorated value is validated or executed
  later. Use this: to close each option combination it's enough to invoke the factory with that
  combination (e.g. a fixture class with one property per combination); you don't need to trigger
  the full validation/execution flow just for coverage, though a real end-to-end case (one positive,
  one negative) confirming the actual behavior — not just coverage — is still worthwhile.
- **Callbacks/parameters with a default value are a separate coverage unit.**
  `function f(cb = (x) => x) {...}` — if every test passes its own `cb`, that default identity
  function stays at 0% function-coverage even if `f` is at 100%. To close it, add a case that
  invokes `f` without that optional argument.
- **Decorators/DI registration**: to cover "invalid class" or "default options," you don't need to
  spin up the whole framework — invoke the decorator as a plain function against a minimal class
  and verify the effect (expected exception, or the config object that ends up registered via a spy
  on the real container/registry method).
- **Guards/middlewares that short-circuit the flow**: register them with the framework's real
  public API (don't mock the whole container) and then invoke the exported entry point directly
  with a minimal simulated context. Avoid touching fixtures shared across several tests — use
  classes and registrations that are new and local to the test.
- **You can't reassign named function exports** (read-only ESM bindings). You CAN reassign
  object/class methods (useful for manual spy/stub with save-original + restore in `finally`, or
  with the test framework's `spy`/`stub` helper if one exists).
- **Safety when forcing a "use the default value/location" fallback branch:** never let that
  fallback's side effect land on a real project path by accident. Safe pattern: stub only the
  primitive that decides the branch (`cwd`, config resolution, fetch) and explicitly pass any
  file read/write parameter to a temp folder — that way the branch's outcome never depends on the
  real environment "luckily" matching something safe. Close every test of this kind with
  `try/finally` restoring the stub and deleting the temp folder, and at the end of the batch run
  `git status`/diff to confirm no real repo file changed.
- **Trust the actual compiler/test-runner run over any live IDE/LSP diagnostic** when they
  contradict the real result — in the sessions this was learned from, live diagnostics were
  occasionally inconsistent with the real file.

## Phase 4 — Verification

- Per new file: run just that file (fast) and confirm with the raw branch data (Phase 0.4) that the
  branch you were after came out green — don't assume it from the test's "ok." Then formatter +
  linter if the project has them configured as a gate.
- Every 5-8 files, or after finishing a themed batch: full run + coverage, and repeat the Phase 0.2
  diff to confirm no file is still invisible.
- If you touched production code (fixing a bug found in Phase 2), run the full suite before and
  after the fix to confirm nothing that depended on the broken behavior breaks.
- Before declaring done: re-extract the list of files below the project's "green" threshold (don't
  just look at the colored summary — use the same raw data from Phase 0.4) and confirm each one
  still there has an explicit Phase 2 reason, not that it was simply not reviewed. If something
  dropped in score versus a previous run, don't assume a regression right away — first check
  whether that file simply hadn't been triaged at the branch level yet (see Phase 0.4).
- If the environment allows it, run `git status`/diff on files that are NOT test or doc files, to
  confirm no forced fallback wrote over something real.
- At the end: one full run, a before/after numbers summary, and a short list of what was left out
  with the reason (one line per item, not an essay).

## Expected report format (to avoid wasting tokens narrating)

```
Coverage: X% branch / Y% function / Z% line (before: X0/Y0/Z0)
N new tests, M total, all passing.

Bugs found via coverage (if any — report separately, not as a test gap):
- file.ts:line — <which branch should trigger and why it doesn't>. Should I fix it?

Closed:
- file.ts: <what branch/gap was closed, in one sentence>

Out of scope (with reason):
- file.ts:line — <dead code | requires live infra | env-dependent with no injection point>
```

---
name: jsdoc-jsr-audit
description: Audits existing JSDoc for accuracy (not just coverage) and drives "deno doc --lint" to zero on Deno/JSR packages — finds doc copy-pasted between sibling symbols, stale defaults/throws, and works through the cycle of exporting private types until the entrypoint is clean. Use it on any Deno package with a deno.json(c) and "exports".
---

Use this as the initial prompt on any Deno/JSR package (`deno.json(c)` with `"exports"`) that
already has JSDoc but where you want to (a) verify it's **correct and not stale**, not just
"present," and/or (b) drive `deno doc --lint` errors to zero (the public-type coverage JSR requires
for a 100 score, and so the generated docs on jsr.io have no gaps).

This task has TWO independent phases you can run separately or together:
- **Phase A — Accuracy audit**: find JSDoc that lies or went stale.
- **Phase B — Zero doc-lint**: export/document whatever JSR requires so `deno doc --lint` ends up
  at 0 errors.

## Golden rule (token savings)

- Run `deno doc --lint <entrypoint>` ONCE, redirect it to a temp file, and strip ANSI with
  `sed -E 's/\x1b\[[0-9;]*m//g'`. Do all the triage against that file with `grep -oE`/`awk`, don't
  re-read the command output over and over.
- Group by rule BEFORE touching anything: `grep -oE "error\[[a-z-]+\]" | sort | uniq -c`. This gives
  you the real size of the problem (typically: `private-type-ref` >> `missing-jsdoc` >
  `missing-return-type`) and keeps you from attacking random files.
- Fix `missing-jsdoc`/`missing-return-type` first (mechanical, no cascade effects) and leave
  `private-type-ref` for last (it DOES cascade — see Phase B).
- After each batch of changes, re-run the full doc-lint once and compare the total count against
  the previous run. Don't explain every individual error in the chat; summarize
  "N→M errors, these remain by category."
- Before reporting "no regressions," compare against the baseline with `git stash` (run the lint
  with the changes stashed, note the number, run `git stash pop`). A live LSP/diagnostic can lie;
  the actual command (`deno check`, `deno doc --lint`, `deno test`) is the source of truth.

## Phase A — Accuracy audit of existing JSDoc (not just coverage)

The goal isn't "it has a comment," it's "the comment is true today." A doc with an invented or
stale `@throws`, default, or behavior claim is worse than no doc at all.

1. **Identify the real public surface**: read the entrypoint file (`mod.ts`/`index.ts`) and list
   ALL re-exported symbols. That list is your scope — don't audit internal code a package consumer
   never sees.
2. **Split into thematic categories** (base/abstract classes, decorators, utils/constants, plain
   types) and launch one `Explore` agent per category IN PARALLEL (a single message with several
   invocations), each with explicit instructions to:
   - Compare every doc claim (parameters, `@returns`, `@throws`, defaults, execution order, "this
     throws X," "this extends Y") against the real implementation, line by line.
   - Report ONLY with high confidence and with `file:line` — no speculation.
   - Return a short report (400–500 words), not a transcription of the code.
   - Be told explicitly what NOT to re-report if you already know of 1-2 bugs you found yourself
     (avoid 4 agents reporting the same thing).
3. **Real bug patterns that show up constantly** (learned from real sessions, look for these
   specifically):
   - Doc copy-pasted between sibling symbols (e.g. `Guard`'s doc with `Pipe`'s prose, or vice
     versa) — the symptom is that two symbols with different behavior have the EXACT same sentence.
   - `@returns`/`@throws` describing a stale value because the signature changed (`grep` the real
     signature, don't trust the doc to tell you what it returns).
   - Execution-order claims ("this applies after X") that are actually backwards — verify by
     reading the code that orchestrates the call (the real "pipeline"), not the decorator's own
     doc.
   - A class decorator (`@Controller`, `@Resolver`, etc.) that validates/throws if the class
     doesn't extend the right base, but that's NOT documented with `@throws`.
   - A function/decorator overload with a field marked optional in the doc but required in the
     real type (or vice versa) — compare `@param [x]` against the real TS signature.
   - A public type whose doc lists fields that no longer exist, or that omits new fields — compare
     the `@property`/prose list against the real `type`/`interface` keys.
   - Real code bugs discovered while auditing docs (don't assume the only bugs are text bugs): a
     `.map()` missing a `return` inside a `Promise.all` (drops awaits/rejections), a type overload
     that requires a field none of its sibling decorators require.
4. **When applying a real code fix found during the audit** (not just a doc fix): run the affected
   area's test suite before and after, and confirm the fix doesn't change observable behavior
   beyond the fixed bug.

## Phase B — Zero `deno doc --lint` errors

### The 3 typical categories and how to fix them

1. **`missing-jsdoc`** — an exported symbol (including methods/constructors/getters of an exported
   class, and even TS `private` members — JSR still sees them because `private` is compile-time
   only) with no comment. Fix: one line of JSDoc explaining what it does, not what's obvious from
   the name.
2. **`missing-return-type`** — an exported function (including `const fn: SomeType = (x) => {...}`)
   with no explicit return annotation on the function expression itself, even if the variable's
   type already implies it. Fix: annotate the return directly on the function
   (`(x): void => {}`, `(x): Promise<void> => {}`).
3. **`private-type-ref`** — a public symbol references a type JSR can't resolve because it's NOT
   reachable from the entrypoint (`mod.ts`). Important: this happens EVEN IF the type already has
   `export` in its own file — JSR also requires it to be re-exported from the entrypoint. And it
   happens doubly if the type doesn't even have `export` in its source file (you have to add it).

### Iterative process for `private-type-ref` (the one with real volume)

1. Extract the unique list of "private" types referenced:
   `grep "error\[private-type-ref\]" | grep -oE "references private type '[^']+'" | sed ... | sort -u`
2. For each unique type, find its source file (`grep -rn "^export type X\|^type X"`). If it has no
   `export`, add it right there (don't duplicate or redefine it elsewhere).
3. Group by source file and add an `export type { A, B, C } from '...'` block in the entrypoint,
   alphabetically ordered within the block so it's easy to maintain.
4. **Cases that aren't a simple loose type — use judgment, don't automate blindly**:
   - Internal abstract base classes that are the real superclass of already-public classes (e.g. a
     public class `extends InternalBase` without `InternalBase` being exported): export them as
     `export type { InternalBase } from '...'` (type-only, no need for the value if nobody's going
     to instantiate it directly — it's abstract). This is the correct fix, not a patch: if a
     consumer extends the public class, they inherit members from that base and need to be able to
     name it.
   - An internal class with no `export` at all (e.g. `class Foo {}` only used as the type of an
     exported singleton, `const publicThing: Readonly<Foo> = ...`): add `export class Foo` in the
     source file; it's a safe change (doesn't change runtime, only makes the type nameable).
   - Types coming from a third-party library (`npm:package`) referenced by one of your public
     types: do NOT chase this by exporting the external package's type. It usually opens up a graph
     of that library's own internal types (generics with their own non-exported defaults) that you
     don't control and that isn't worth fixing. Leave that ONE error as a documented exception and
     move on. How to confirm this: if re-exporting the external type makes the error count go UP
     instead of down, that's the signal you've stepped into a third-party package's internal type
     graph — revert that specific export.
5. **Expected cascade, not your fault**: every type that goes from private to public can:
   - Need its own JSDoc (wasn't required before because it was private) → new batch of
     `missing-jsdoc`, fix it the same way as in Phase B.1.
   - Reference yet ANOTHER unreachable type → new batch of `private-type-ref`.
   Iterate: batch fix → `deno check <entrypoint>` (no typos/broken exports) → `deno doc --lint` →
   repeat. 3-5 rounds is normal for a package with ~70 initial errors.
6. **Watch out for batch text editing**: if you use `sed`/python to insert `export` before a
   `type X = ...` in several files at once, verify immediately with `deno check` afterward — an
   `old_string` that's too short can match as a substring inside a line that already had `export`
   and duplicate it (`export /** doc */\nexport type X = ...`), which is a silent `SyntaxError`
   until you run the check.

## Final verification (don't skip it)

```bash
deno fmt --check <paths>
deno lint <paths>
deno check <entrypoint>
deno test --allow-all   # or the project's runner
deno doc --lint <entrypoint>   # must drop to 0, or to documented third-party exceptions
deno publish --dry-run --allow-dirty   # confirms "Checking for slow types..." with no warnings
```

To confirm you didn't break anything pre-existing (and not just that "the number didn't go up"):
```bash
git stash && deno doc --lint <entrypoint> 2>&1 | tail -3 && git stash pop
```

## Expected report format (to avoid wasting tokens narrating)

```
Doc-lint: X errors → Y errors (categories: private-type-ref A→B, missing-jsdoc C→D, missing-return-type E→F)
Documented exceptions (deliberately not fixed):
- <type> — references an internal type of <npm package> not exportable without dragging in its internal graph.

Real code bugs found during the audit (not just doc):
- file.ts:line — <what was wrong and what was fixed, one sentence>

Docs fixed for being stale (show only the most important ones, not all 30):
- file.ts:line — <what it said vs. what's true now, one sentence>

Verification: fmt/lint/check/test OK, doc-lint with no regression (confirmed via git stash).
```

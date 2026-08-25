---
name: jsdoc-jsr-audit
description: Audits existing JSDoc for accuracy (not just coverage) and drives "deno doc --lint" to zero on Deno/JSR packages — finds doc copy-pasted between sibling symbols, stale defaults/throws, and works through the cycle of exporting private types until the entrypoint is clean. Use it on any Deno package with a deno.json(c) and "exports".
---

Use this as the initial prompt on any Deno/JSR package (`deno.json(c)` with
`"exports"`) that already has JSDoc but where you want to (a) verify it's
**correct and not stale**, not just "present," and/or (b) drive
`deno doc --lint` errors to zero (the public-type coverage JSR requires for a
100 score, and so the generated docs on jsr.io have no gaps).

This task has TWO independent phases you can run separately or together:

- **Phase A — Accuracy audit**: find JSDoc that lies or went stale.
- **Phase B — Zero doc-lint**: export/document whatever JSR requires so
  `deno doc --lint` ends up at 0 errors.

**If asked to "apply this skill" without the caller specifying a phase, run
BOTH.** Phase B's mechanical fixes are the easy part and finish fast; skipping
straight to "done" after only Phase B means the accuracy audit — the part that
actually catches lying docs — never ran. Don't report Phase B's result as if it
were the complete audit. If you do scope down to one phase (time/size
constraints), say so explicitly in the report — never omit it silently.

## Golden rule (token savings)

- Run `deno doc --lint <entrypoint>` ONCE, redirect it to a temp file, and strip
  ANSI with `sed -E 's/\x1b\[[0-9;]*m//g'`. Do all the triage against that file
  with `grep -oE`/`awk`, don't re-read the command output over and over.
- Group by rule BEFORE touching anything:
  `grep -oE "error\[[a-z-]+\]" | sort | uniq -c`. This gives you the real size
  of the problem (typically: `private-type-ref` >> `missing-jsdoc` >
  `missing-return-type`) and keeps you from attacking random files.
- Fix `missing-jsdoc`/`missing-return-type` first (mechanical, no cascade
  effects) and leave `private-type-ref` for last (it DOES cascade — see Phase
  B).
- After each batch of changes, re-run the full doc-lint once and compare the
  total count against the previous run. Don't explain every individual error in
  the chat; summarize "N→M errors, these remain by category."
- Before reporting "no regressions," compare against the baseline with
  `git stash` (run the lint with the changes stashed, note the number, run
  `git stash pop`). A live LSP/diagnostic can lie; the actual command
  (`deno check`, `deno doc --lint`, `deno test`) is the source of truth.

## Phase A — Accuracy audit of existing JSDoc (not just coverage)

The goal isn't "it has a comment," it's "the comment is true today." A doc with
an invented or stale `@throws`, default, or behavior claim is worse than no doc
at all.

**Language and tone**: every JSDoc comment follows `documentation-voice` —
apply it during this same pass, flagged as a Phase A finding like any
other, not as a separate audit.

1. **Identify the real public surface**: read the entrypoint file
   (`mod.ts`/`index.ts`) and list ALL re-exported symbols. That list is your
   scope — don't audit internal code a package consumer never sees.
2. **Split into thematic categories** (base/abstract classes, decorators,
   utils/constants, plain types) and launch one `Explore` agent per category IN
   PARALLEL (a single message with several invocations), each with explicit
   instructions to:
   - Compare every doc claim (parameters, `@returns`, `@throws`, defaults,
     execution order, "this throws X," "this extends Y") against the real
     implementation, line by line.
   - Report ONLY with high confidence and with `file:line` — no speculation.
   - Return a short report (400–500 words), not a transcription of the code.
   - Be told explicitly what NOT to re-report if you already know of 1-2 bugs
     you found yourself (avoid 4 agents reporting the same thing).
3. **Real bug patterns that show up constantly** (learned from real sessions,
   look for these specifically):
   - Doc copy-pasted between sibling symbols (e.g. `Guard`'s doc with `Pipe`'s
     prose, or vice versa) — the symptom is that two symbols with different
     behavior have the EXACT same sentence.
   - `@returns`/`@throws` describing a stale value because the signature changed
     (`grep` the real signature, don't trust the doc to tell you what it
     returns).
   - Execution-order claims ("this applies after X") that are actually backwards
     — verify by reading the code that orchestrates the call (the real
     "pipeline"), not the decorator's own doc.
   - A class decorator (`@Controller`, `@Resolver`, etc.) that validates/throws
     if the class doesn't extend the right base, but that's NOT documented with
     `@throws`.
   - A function/decorator overload with a field marked optional in the doc but
     required in the real type (or vice versa) — compare `@param [x]` against
     the real TS signature.
   - A public type whose doc lists fields that no longer exist, or that omits
     new fields — compare the `@property`/prose list against the real
     `type`/`interface` keys.
   - Real code bugs discovered while auditing docs (don't assume the only bugs
     are text bugs): a `.map()` missing a `return` inside a `Promise.all` (drops
     awaits/rejections), a type overload that requires a field none of its
     sibling decorators require.
4. **When applying a real code fix found during the audit** (not just a doc
   fix): run the affected area's test suite before and after, and confirm the
   fix doesn't change observable behavior beyond the fixed bug.

## Phase B — Zero `deno doc --lint` errors

### The 3 typical categories and how to fix them

1. **`missing-jsdoc`** — an exported symbol (including
   methods/constructors/getters of an exported class, and even TS
   `private`/`protected` members — JSR still sees them because `private` is
   compile-time only) with no comment. Fix: one line of JSDoc explaining what it
   does, not what's obvious from the name. ⚠️ **`deno doc --lint` does NOT
   reliably flag every undocumented `private`/`protected` member** — some field
   patterns (e.g. `private x = someBoundFunction`, or a field followed only by a
   trailing `// line comment` instead of a real `/** */` block) silently pass
   the lint even with zero real documentation. Reaching 0 lint errors is
   necessary but NOT sufficient proof that every private/protected member is
   documented — do the manual sweep in "Final verification" below regardless of
   the lint count. (`#`-prefixed real private fields are invisible to
   JSR/deno_doc and never need docs — this only applies to TS
   `private`/`protected` keyword members.)
2. **`missing-return-type`** — an exported function (including
   `const fn: SomeType = (x) => {...}`) with no explicit return annotation on
   the function expression itself, even if the variable's type already implies
   it. Fix: annotate the return directly on the function (`(x): void => {}`,
   `(x): Promise<void> => {}`).
3. **`private-type-ref`** — a public symbol references a type JSR can't resolve
   because it's NOT reachable from the entrypoint (`mod.ts`). Important: this
   happens EVEN IF the type already has `export` in its own file — JSR also
   requires it to be re-exported from the entrypoint. And it happens doubly if
   the type doesn't even have `export` in its source file (you have to add it).

### A 4th failure mode `deno doc --lint` never shows you: cross-entrypoint re-export duplication

This ONLY affects packages with **2+ entrypoints** in `deno.json`'s `"exports"`
map where one entrypoint re-exports another via `export * from '...'` (e.g. a
root `mod.ts` that does `export * from './database/mod.ts'` and
`export * from './cache/mod.ts'`). Single-entrypoint packages
(`"exports": { ".": "./mod.ts" }`) cannot hit this, no matter how many named
re-exports `mod.ts` has — verified empirically: `@zanix/server` (one entrypoint,
dozens of bare `export { X } from '...'` lines with zero local comments) scores
94% because `deno_doc` correctly follows each reference back to `X`'s real JSDoc
when there's only one root to resolve against.

The bug: JSR publishes by feeding **every** declared entrypoint to `deno_doc`'s
`DocParser` together in one call (`analysis.rs`'s `roots` vector = all of
`exports`'s values). When a symbol is reachable through two of those roots at
once — once directly, via a named re-export (`export { X } from './y.ts'`)
sitting IN one of the entrypoint files itself, and again transitively through
another entrypoint's `export *` — `deno_doc` resolves the reference correctly in
the `export *` root's own bucket, but **loses the jsDoc in the
entrypoint-that-contains-the-named- re-export's own bucket**, even though it's
the exact same reference to the exact same declaration. Reproduced in isolation
with 3 minimal files (`source.ts` real decl → `sub.ts` does
`export { doThing } from './source.ts'` → `root.ts` does
`export * from './sub.ts'`): parsing `sub.ts` alone resolves fine; parsing
`root.ts` + `sub.ts` together (matching how JSR parses a multi-entrypoint
package) leaves `doThing` undocumented specifically in `sub.ts`'s own bucket.
JSR's score sums undocumented counts across every entrypoint's bucket, so this
silently drags the total percentage down even though
`deno doc --lint <entrypoint>` on any single entrypoint shows 0 errors and the
real declaration is fully documented.

The workaround: put the doc comment **inside the braces**, directly above the
name, instead of above the whole `export { ... } from '...'` statement — the
declaration then carries its own literal jsDoc and doesn't depend on the (buggy)
cross-root reference resolution at all:

```ts
// ❌ still breaks in the sub-entrypoint's own bucket when another entrypoint `export *`s this file
/** Does X. */
export { X } from "./y.ts";

// ✅ works regardless of cross-entrypoint duplication
export {
  /** Does X. */
  X,
} from "./y.ts";
```

Only sweep for this in packages with multiple `exports` entries where
entrypoints cross-reference each other. Multi-name blocks
(`export { A, B, C } from '...'`) are usually already written with one doc per
line inside the braces, so they're typically unaffected already — the trap is
specifically the single-name, one-line `export { X } from '...'` /
`export type { X } from '...'` shorthand.

Since `deno doc --lint` is silent here, verify locally with the exact method JSR
itself uses (`deno_doc`'s `DocParser`, fed all of `deno.json`'s `exports`
entries in one call — this is what actually reproduces the cross-entrypoint
duplication, not a per-entrypoint `deno doc --json` call):

```bash
deno doc --json $(jq -r '.exports[]' deno.json) > /tmp/doc.json
python3 -c "
import json
d = json.load(open('/tmp/doc.json'))
total = documented = 0
for filedata in d['nodes'].values():
    for s in filedata.get('symbols', []):
        for decl in s.get('declarations', []):
            total += 1
            documented += bool(decl.get('jsDoc'))
print(f'{documented}/{total} = {documented/total*100:.1f}%')
"
```

This reproduces JSR's real percentage almost exactly (confirmed against a live
package where it matched to the decimal) — trust this number over
`deno doc --lint`'s error count when judging whether the _score_ will pass, not
just the lint. To see the actually-published number for a package already on
JSR: `curl -s https://api.jsr.io/scopes/<scope>/packages/<pkg>/score` — note
this value is baked in at publish time per version, not recomputed live, so a
local fix only shows up on JSR after a new version is actually published.

### A 5th failure mode `deno doc --lint` reports as something else entirely: `missing-explicit-type` slow types from schema-validation libraries

`deno doc --lint` and `deno publish --dry-run`'s slow-types checker can both
fire on the exact same line and report it under **two different,
unrelated-looking codes** — `private-type-ref` from `deno doc --lint`,
`missing-explicit-type` from the slow-types checker — because they're answering
different questions ("can I resolve this reference at all" vs. "can I resolve
this reference's type _without running inference_"). Applying
`private-type-ref`'s standard remedy (export/re-export the referenced type — see
step 4 above) does **not** fix the slow-types error, and can even leave it in
place while making the doc-lint error disappear: exporting a value doesn't give
it an explicit type annotation, and an explicit annotation is exactly what the
slow-types checker is asking for.

The classic trigger is a public type derived via inference from a
schema-validation library's builder output — most commonly Zod's
`z.infer<typeof someSchema>` (the same shape applies to Valibot, io-ts, or any
other "build a validator, infer the static type from it" library) — where
`someSchema` itself has no explicit type annotation:

```ts
// schema.ts (internal, never exported)
export const genericSchema = z.object({
  title: z.string(),
  content: z.string(),
});

// typings.ts (public) — this is what actually triggers the error
import type GenericSchema from "./schema.ts";
export type GenericTemplateSchema = z.infer<typeof GenericSchema>;
```

`deno publish --dry-run --allow-dirty` reports this as `missing-explicit-type`
pointing at `genericSchema`'s own declaration — not at the public
`GenericTemplateSchema` alias. Confirm and triage the same way as
`private-type-ref` (own its own pass, don't assume `deno doc --lint` already
covers it):

```bash
deno publish --dry-run --allow-dirty 2>&1 | sed -E 's/\x1b\[[0-9;]*m//g' > /tmp/slow-types.txt
grep -c "error\[missing-explicit-type\]" /tmp/slow-types.txt
grep -B1 -A6 "error\[missing-explicit-type\]" /tmp/slow-types.txt
```

**Cascades further than `private-type-ref` does.** The slow-types checker needs
to fully resolve the schema's _entire_ composed shape, so it also flags
completely unexported locals used _inside_ that schema's construction (a
sub-schema value merged in via `.and()`/spread/composition) — symbols
`deno doc --lint`'s reference-following never even mentions, because they're
never the direct `typeof` target, just an ingredient of it. Don't assume the
list from `deno doc --lint` is a superset; re-derive it from
`deno publish --dry-run` directly.

**The fix is NOT to hand-annotate the schema's own type.** For builder-pattern
validation libraries, the real generic type of a composed schema
(`ZodObject<{...}, ...>` with its full parameter list) is often impractically
verbose to write and re-write by hand on every field change — and doing so
defeats much of the point of using the library. The clean fix is to
**decouple**: keep the schema itself unexported and internal (used only for
runtime parsing, wherever the code actually calls `.parse()`/`.safeParse()`),
and hand-write the public-facing type once, as a plain `interface`/`type`
matching the schema's effective shape field-by-field:

```ts
// typings.ts (public) — no import of the schema at all, no z.infer
/** Data accepted by the `generic` template. Mirrors `schema.ts`'s `genericSchema` shape. */
export type GenericTemplateSchema = {
  title: string;
  content: string;
};
```

This trades a small hand-maintenance cost (update the type by hand if the
schema's fields change) for total freedom in the internal schema — it can use
any library feature, any nesting, any version upgrade, without ever touching the
published package's type-check score again. Verify the new type still matches
the real schema field-by-field before committing to it (read the schema file,
don't guess), and re-run the `grep` above to confirm the count actually dropped
— don't rely on `deno doc --lint` to confirm this category, since it was never
measuring the same thing.

### Iterative process for `private-type-ref` (the one with real volume)

1. Extract the unique list of "private" types referenced:
   `grep "error\[private-type-ref\]" | grep -oE "references private type '[^']+'" | sed ... | sort -u`
2. For each unique type, find its source file
   (`grep -rn "^export type X\|^type X"`). If it has no `export`, add it right
   there (don't duplicate or redefine it elsewhere).
3. Group by source file and add an `export type { A, B, C } from '...'` block in
   the entrypoint, alphabetically ordered within the block so it's easy to
   maintain.
4. **Cases that aren't a simple loose type — use judgment, don't automate
   blindly**:
   - The referenced "private" type is a schema-validation library's builder
     output (Zod, Valibot, etc.) with no explicit type annotation of its own:
     exporting/re-exporting it clears the `private-type-ref` doc-lint error but
     does NOT fix the underlying `missing-explicit-type` slow-types error — see
     "A 5th failure mode" below for the real fix (decouple the public type from
     the schema instead of making the schema reachable).
   - Internal abstract base classes that are the real superclass of
     already-public classes (e.g. a public class `extends InternalBase` without
     `InternalBase` being exported): export them as
     `export type { InternalBase } from '...'` (type-only, no need for the value
     if nobody's going to instantiate it directly — it's abstract). This is the
     correct fix, not a patch: if a consumer extends the public class, they
     inherit members from that base and need to be able to name it.
   - An internal class with no `export` at all (e.g. `class Foo {}` only used as
     the type of an exported singleton,
     `const publicThing: Readonly<Foo> = ...`): add `export class Foo` in the
     source file; it's a safe change (doesn't change runtime, only makes the
     type nameable).
   - Types coming from a third-party library (`npm:package`) referenced by one
     of your public types: do NOT chase this by exporting the external package's
     type. It usually opens up a graph of that library's own internal types
     (generics with their own non-exported defaults) that you don't control and
     that isn't worth fixing. Leave that ONE error as a documented exception and
     move on. How to confirm this: if re-exporting the external type makes the
     error count go UP instead of down, that's the signal you've stepped into a
     third-party package's internal type graph — revert that specific export.
5. **Expected cascade, not your fault**: every type that goes from private to
   public can:
   - Need its own JSDoc (wasn't required before because it was private) → new
     batch of `missing-jsdoc`, fix it the same way as in Phase B.1.
   - Reference yet ANOTHER unreachable type → new batch of `private-type-ref`.
     Iterate: batch fix → `deno check <entrypoint>` (no typos/broken exports) →
     `deno doc --lint` → repeat. 3-5 rounds is normal for a package with ~70
     initial errors.
6. **Watch out for batch text editing**: if you use `sed`/python to insert
   `export` before a `type X = ...` in several files at once, verify immediately
   with `deno check` afterward — an `old_string` that's too short can match as a
   substring inside a line that already had `export` and duplicate it
   (`export /** doc */\nexport type X = ...`), which is a silent `SyntaxError`
   until you run the check.

## Final verification (don't skip it)

```bash
deno fmt --check <paths>
deno lint <paths>
deno check <entrypoint>
deno test --allow-all   # or the project's runner
deno doc --lint <entrypoint>   # must drop to 0, or to documented third-party exceptions
deno publish --dry-run --allow-dirty   # must report 0 errors, not just "no warnings" — see below
```

Don't treat the `deno publish --dry-run` line as a rubber-stamp confirmation —
it's a real, independent diagnostic pass (see "A 5th failure mode" above). Any
`error[missing-explicit-type]` it reports is a genuine slow-types problem
`deno doc --lint` may report as something else entirely (or not report at all),
and needs its own triage pass, not just a glance at the exit status.

`deno doc --lint` at 0 is necessary but not sufficient for the JSR score itself
— also run the combined `deno doc --json` + percentage script from "A 4th
failure mode" above against ALL of `deno.json`'s `exports` entries, and confirm
it's ≥80% (ideally 100%) before considering the doc work done.

To confirm you didn't break anything pre-existing (and not just that "the number
didn't go up"):

```bash
git stash && deno doc --lint <entrypoint> 2>&1 | tail -3 && git stash pop
```

**Mandatory manual sweep for `private`/`protected` members** — run this even
when `deno doc --lint` is already at 0, since the lint misses real gaps (see the
⚠️ under `missing-jsdoc` above). Scan every class in the touched files (or the
whole public surface, on a full audit) for a `private`/`protected` member whose
preceding non-blank line isn't a closing `*/` of a JSDoc block:

```bash
python3 - <<'EOF'
import re, glob
files = [f for pat in ['src/**/*.ts', 'mod.ts'] for f in glob.glob(pat, recursive=True)
          if '@tests' not in f and 'node_modules' not in f]
priv_re = re.compile(r'^\s*(private|protected)\s+(readonly\s+)?(get\s+|set\s+)?(#?\w+)')
for fname in sorted(set(files)):
    lines = open(fname).readlines()
    for i, line in enumerate(lines):
        m = priv_re.match(line)
        if not m or (m.group(3) or '').strip() == 'set':
            continue  # setters pair with a documented getter, no separate doc needed
        j = i - 1
        while j >= 0 and lines[j].strip() == '':
            j -= 1
        prev = lines[j].strip() if j >= 0 else ''
        if not prev.endswith('*/'):
            print(f'{fname}:{i+1}: {line.rstrip()}')
EOF
```

Adjust the glob to the project's actual layout. Any line printed is a real gap —
add a one-line `/** ... */` immediately above it (a trailing `// comment` does
NOT count, replace it). Re-run the script after fixing until it prints nothing,
then re-run the full verification block above.

## Expected report format (to avoid wasting tokens narrating)

```
Doc-lint: X errors → Y errors (categories: private-type-ref A→B, missing-jsdoc C→D, missing-return-type E→F)
Slow types (deno publish --dry-run): X errors → Y errors (missing-explicit-type)
Documented exceptions (deliberately not fixed):
- <type> — references an internal type of <npm package> not exportable without dragging in its internal graph.

Real code bugs found during the audit (not just doc):
- file.ts:line — <what was wrong and what was fixed, one sentence>

Docs fixed for being stale (show only the most important ones, not all 30):
- file.ts:line — <what it said vs. what's true now, one sentence>

Verification: fmt/lint/check/test OK, doc-lint with no regression (confirmed via git stash).
JSR symbol-documentation percentage: X% → Y% (via the combined deno doc --json script).
```

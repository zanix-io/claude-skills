---
name: docs-readme-audit
description: Makes a Deno/JSR package's README, docs/, and CHANGELOG complete, coherent, and professional in one exhaustive pass — symbol cross-check against real exports, link/anchor integrity, full exports→docs coverage, and optional validation against a real production consumer. Use it when asked to improve/complete/professionalize a package's documentation.
---

Use this as the initial prompt on any Deno/JSR package (`deno.json(c)` with `"exports"`) when you
want to leave its README, `docs/`, and `CHANGELOG.md` complete, coherent, and professional in one
pass — without the "is it done yet? are you sure nothing's missing?" back-and-forth that triggers
repeated re-reads. This skill exists precisely to avoid that token waste: do ALL the checks below
in the first pass, not one per turn.

You can run this as a full task, or invoke just the Phase you need (e.g. "just Phase 4" for a
coherence check without touching content).

## Golden rule (token savings)

- **One-shot exhaustiveness, not iterative.** The user shouldn't have to ask "is anything else
  missing?" three times before real issues surface. Before saying "done," run ALL the checks in
  Phase 1 and Phase 4 in a single sweep — they're cheap (grep/scripts, not manual re-reading) and
  each one has historically found real bugs, not cosmetic ones.
- **Verify programmatically, not by eyeballing.** A broken link, a misspelled symbol name in an
  example, or an example that doesn't compile against the real signature, are found with a 10-line
  script (see Phase 4), not by re-reading the file five times. Write the script once, run it, fix
  what it reports. This includes verifying that **anchors** (`#section`) resolve using the
  renderer's real slug algorithm (see Phase 4) — don't assume it collapses spaces/symbols the same
  way your first intuition does; a heading with "&" or another special character can produce a
  double-hyphen slug that breaks the link if you compute it wrong.
- **Don't document what you can't verify against the real source.** If you're about to write a
  usage example, check it against the real implementation (types, signature, behavior) or against
  an existing test/fixture — never "based on what sounds plausible." Real sessions found actual
  code bugs (not just doc bugs) precisely by verifying instead of assuming.
- **Summarize, don't transcribe.** Report findings as a short list with `file:line`, don't dump the
  whole file into the chat at every step.
- **Let `deno fmt` do the wrapping, not you.** If `deno.json(c)` has `"fmt"` with `proseWrap` and
  doesn't exclude `.md`, write the markdown in natural paragraphs and run `deno fmt` at the end —
  don't hand-wrap lines, it's slower and less consistent.
- **Before "fixing" something that looks like a leftover artifact from a previous edit, confirm it
  actually is one.** If the user or a previous edit added content you don't fully recognize, don't
  assume it's your mistake or an accident — ask or verify before deleting it. A `git diff` showing
  two adjacent changes in the same region doesn't prove they're the same change.

## Phase 0 — Scope and starting point

1. Read the current `README.md` in full, the `CHANGELOG.md`, and list what already exists under
   `docs/`.
2. Read the entrypoint file (`mod.ts`/`index.ts`) and extract the COMPLETE list of re-exported
   symbols (classes, functions, constants, types). This is your ground truth surface — everything
   else is measured against this list.
3. Only ask about scope if it's genuinely ambiguous (e.g. "does the README already have a
   format/tone I should respect, or can I restructure freely?"). If the user says "don't change the
   format unnecessarily," treat it as: you can reorder/trim/segment content and fix bugs, but don't
   rewrite the tone or the section structure that already works.

## Phase 1 — Exhaustive README audit (all in one pass)

Don't wait for the user to ask "is it complete?" to find these — look for all of them now:

1. **Symbol cross-check**: extract with a script (regex over code blocks with
   `import { A, B, C } from '...'`) every identifier mentioned in the README's examples, and compare
   it against the real export list from Phase 0. Any name that doesn't match is a real typo (e.g.
   `SomeAsyncmqProvider` vs `SomeAsyncMQProvider` — a single character, invisible at a glance, found
   only by the automated cross-check).
2. **Every code example must reflect the real API**: if an example calls `object.method(x)`, verify
   against the real signature what `x` actually is — a common mistake is passing the type/name of
   something instead of the identifier the method actually expects (e.g. `manager.start('rest')`
   when `start()` expects the ID that `create()` returned, not the type string). If you find a wrong
   factual claim (a bad env var name, a wrong default), grep that exact wrong string across the
   source's own JSDoc too, not just the README — a README is often partly copy-pasted from JSDoc, so
   the same lie can have one shared origin in two (or three) places, and fixing only the README
   leaves the underlying doc-in-source still wrong for anyone reading `deno doc` directly.
3. **Links and anchors**: every internal `[text](path)` must resolve to a real file, and every
   `[text](path#section)` must resolve to a real heading in that file — verify both with a script
   (see Phase 4), not visually. Also check badges (shields.io, etc.): the badge link's URL sometimes
   doesn't match the image URL (typos like `@org/project` vs `@org/project2`).
4. **Table of contents vs. real headings**: count the document's `##` headings and compare them 1:1
   against the ToC — a new heading (e.g. `## Changelog`) that was never added to the ToC is a
   common, silent bug.
5. **Flagship example = the real idiomatic pattern, not the "escape hatch"**: if the project has a
   recommended pattern (decorators, builder, etc.) and also a low-level API for advanced cases, the
   "Basic Usage"/"Quickstart" example must show the recommended pattern — not the low-level one.
   Verify this explicitly by comparing the flagship example against what the rest of the
   documentation (and the real test code) treats as "the normal way to do it."
6. **Real redundancy vs. intentional duplication**: a long import/feature catalog, or a full table
   (e.g. of environment variables), that repeats content already covered word-for-word in another
   section or in `docs/`, should be trimmed to a compact version with a link — don't leave it
   duplicated in two places. A short example repeated as a teaser in two spots (README + guide) IS
   fine; a full duplicated catalog or table is NOT — check this explicitly by comparing every README
   table against the ones in `docs/`, not only when writing new content.
6b. **A README that's already 100% accurate and covers every export is not the same as a README
   that's well-organized — segment regardless of whether coverage already technically passes.** If
   there's no `docs/` yet, that's not evidence the README doesn't need it: check length/structure on
   their own terms (a self-contained section long enough to have its own sub-numbering, or one that
   could stand alone as a topic, per 1.6/1.7) and segment into Phase 2 whenever those triggers fire.
   Don't let "Phase 2's coverage requirement is already satisfied inside the README" talk you out of
   doing Phase 2's actual segmentation work — coverage and organization are different bars.
7. **Logical order**: install/setup steps that ended up "loose" between two unrelated sections
   should move next to the Installation section. An unnumbered section that interrupts a numbered
   sequence (1, 2, [no number], 3) breaks the reading flow — move it to the end.
8. **Style consistency**: emojis in headings — either used on every heading of the same level, or
   on none; not mixed. Headings ending in a colon (`## Title:`) inconsistent with the rest of the
   document. Warning/note blockquotes with no marker (ℹ️/⚠️) when the rest of the document does use
   one for the same kind of notice. Filler sentences ("see the full documentation for more
   examples") right before a section that already does exactly that with real links — delete them.
9. **Compatibility/requirements note**: if the CHANGELOG or another internal source mentions a
   version requirement (e.g. "compatible with Deno 2.9"), and that requirement isn't visible
   anywhere for someone installing for the first time, add it under Installation.
10. **Don't invent badges**: if you're about to suggest a CI/coverage badge, first verify a real
    workflow backs it (`.github/workflows/`) — if it doesn't exist, don't add it or suggest it as if
    it did.
11. **ASCII diagrams vs. prose**: if there's an architecture diagram, confirm what it draws (boxes,
    arrows connecting them) doesn't contradict what the text right next to it explicitly claims — a
    diagram that only draws 2 of 4 connections mentioned in the text is a real source of confusion,
    not just a cosmetic detail.
12. **JSR "Overview" tab vs. `README.md` (JSR packages only)**: if the root entrypoint (`.` in
    `exports`) has a `@module`-tagged JSDoc comment, JSR's Overview tab shows THAT comment instead
    of the real `README.md` — all the README work above can go live and still not be what visitors
    see. Don't "fix" this by removing the `@module` tag: JSR's separate "Has module docs in all
    entrypoints" score item (Phase 4.6) requires every entrypoint, including the root, to have one
    — removing it trades a display problem for a score regression. The supported fix is the
    package's jsr.io **Settings → Readme Source → "Readme"** toggle, which forces README display
    regardless of `@module`. That's a manual step on jsr.io requiring the owner's authenticated
    session — flag it to the user with exact instructions, don't try to route around it in code.

## Phase 2 — Full coverage in `docs/` (exports → documentation mapping)

1. Using the export list from Phase 0, group them by the domain's natural thematic category (for a
   server library: Handlers, Middlewares, Dependency Injection, Configuration, Errors, Utilities —
   adjust the categories to the real domain of the library you're documenting).
2. For each exported symbol, verify it has a documented home in some guide. If it doesn't, decide:
   is it important/used enough to deserve its own section, or is it low-level internal plumbing
   used mostly by the framework itself (e.g. a function that only calls itself during the internal
   lifecycle)? Document the former with a full example; the latter with a compact reference table
   (one row per symbol, no lengthy example) — don't give both the same weight.
3. **Segment, don't duplicate**: when you trim content from the README (Phase 1.6), the trimmed
   detail moves to `docs/`, it isn't lost. Verify with the same Phase 1.1 script that the moved
   content is still accurate in its new home.
4. **Link both ways**: every new guide under `docs/` must have a "See also" section linking to
   related guides, and the README must link to all of them from its Documentation section.
5. **Be explicit about where each piece in an example comes from**: if an example uses a concrete
   class that isn't part of this library (it comes from a sibling library or the user writes it
   themselves), say so explicitly — don't let the reader assume it comes from the library you're
   documenting. Verify the real origin (a real import in a consumer, or the source itself) before
   asserting it.
6. **Update or extend an existing guide before creating a new one** when the new content
   thematically belongs to a guide that's already written — avoid fragmenting into too many small
   files.
7. **Also verify defaults and cross-symbol constraints for related symbols**: if you document three
   variants of the same concept (e.g. three decorators that accept the same options), confirm in
   the code that all three really accept the same values — an enum value valid for two but excluded
   by type for the third is an easy gap to miss if you only look at one example at a time.
8. **One capability, multiple exposure surfaces — document all of them, not just the first you
   find**: the same underlying operation is often reachable through more than one surface (e.g. a
   schema getter, a standalone exported function, AND a static method attached to a bound
   model/instance for the same protection/transform logic). Finding and documenting one surface
   (the getter, say) doesn't mean the others are covered — explicitly grep the type definitions
   (`SchemaStatics`, or whatever the bound-instance's static-method type is called) for methods that
   duplicate a capability you already wrote up under a different name/form, and cover each real
   surface with its own short example, cross-linked to the others.

## Phase 3 — CHANGELOG and version

0. **Check the CHANGELOG isn't already stale before adding to it**: compare its most recent version
   entry against the real `version` field in `deno.json(c)`. If the package is several versions
   ahead of what the CHANGELOG shows, there's undocumented release history — find the commit/tag for
   the last-logged version (`git log --oneline -- deno.json(c)` to track each version bump), diff
   forward from there to HEAD in per-version chunks, and backfill one entry per real released
   version (with its actual date from the commit) before writing today's new entry on top. Don't
   skip this because it's tedious — a CHANGELOG stuck 6 versions behind the real package version is
   a bigger real-world problem than any single missing line in the README.
1. Add a new entry following the existing format (usually Keep a Changelog:
   Added/Changed/Fixed/Removed/Deprecated/Security). Group by those categories, not by
   file-touched.
2. If real code bugs were fixed during the audit (not just text), state them under `Fixed` with a
   sentence explaining the real symptom, not just "fixed X."
3. Decide the version bump based on real impact: docs/new type exports only, nothing broken = minor;
   behavior bugs fixed = patch or minor depending on what the project has been doing (check the
   CHANGELOG history to infer the convention it already follows). If the session's work hasn't been
   published/committed yet, don't create multiple version bumps for different rounds of the same
   session — everything fits in a single version entry.
4. Update the real `version` field in `deno.json(c)` to match the new CHANGELOG entry.

## Phase 4 — Automated verification (run it all together, not piecemeal)

```bash
# 1. Formatting
deno fmt --check README.md CHANGELOG.md docs/*.md

# 2. Internal link integrity (once, for ALL .md files at a time)
python3 - <<'EOF'
import re, os
files = ['README.md'] + [f'docs/{f}' for f in os.listdir('docs')]
ok = True
for f in files:
    content = open(f).read()
    base = os.path.dirname(f)
    for m in re.finditer(r'\]\(([^)]+)\)', content):
        link = m.group(1)
        if link.startswith('http') or link.startswith('#'):
            continue
        target = os.path.normpath(os.path.join(base, link.split('#')[0]))
        if not os.path.exists(target):
            print(f"BROKEN in {f}: {link} -> {target}")
            ok = False
print("All internal links resolve." if ok else "Some broken.")
EOF

# 3. Anchor integrity (#section) against real headings — use the real slug algorithm
#    (github-slugger: lowercase, strip punctuation except spaces/hyphens, EACH space -> ONE
#    hyphen WITHOUT collapsing runs; an "&" can leave two spaces in a row -> "--")
python3 - <<'EOF'
import re, os

def slugify(h):
    h = h.lower().strip()
    h = re.sub(r'[`*]', '', h)
    h = re.sub(r'[^\w\s-]', '', h)
    h = re.sub(r'\s', '-', h)  # no '+': doesn't collapse consecutive spaces
    return h

files = ['README.md'] + [f'docs/{f}' for f in os.listdir('docs')]
headings = {f: {slugify(h) for h in re.findall(r'^#{1,6}\s+(.+)$', open(f).read(), re.MULTILINE)} for f in files}

ok = True
for f in files:
    content = open(f).read()
    base = os.path.dirname(f)
    for m in re.finditer(r'\]\(([^)]+)\)', content):
        link = m.group(1)
        if '#' not in link or link.startswith('http'):
            continue
        path, frag = link.split('#', 1)
        target_file = f if not path else os.path.normpath(os.path.join(base, path))
        if target_file in headings and frag not in headings[target_file]:
            print(f"ANCHOR MISMATCH in {f}: #{frag} -> {target_file}")
            ok = False
print("All anchors resolve." if ok else "Some anchors mismatched.")
EOF

# 4. Symbol cross-check in examples vs. real exports (adapt the regex to the language/package)
python3 - <<'EOF'
import re
readme = open('README.md').read()
mod = open('mod.ts').read()  # or the real entrypoint
blocks = re.findall(r"import\s*\{([^}]+)\}\s*from\s*'jsr:@scope/pkg", readme)
names = {n.strip() for b in blocks for n in b.split(',') if n.strip()}
exported = set(re.findall(r'export\s*\{\s*([^}]+)\}', mod))
exported_names = set()
for block in exported:
    for item in block.split(','):
        item = item.strip()
        exported_names.add(item.split(' as ')[1].strip() if ' as ' in item else item)
missing = [n for n in names if n not in exported_names]
print("Missing from real exports:", missing)
EOF

# 5. Types, tests, and (if JSR) doc-lint + slow-types
deno check <entrypoint>
deno test --allow-all
# Lint ALL entrypoints from deno.jsonc's exports map together, not just the root — linting
# only mod.ts undercounts errors that live in the other entrypoint files.
deno doc --lint <entrypoint1> <entrypoint2> ...  # must be 0, or only documented third-party exceptions
deno publish --dry-run --allow-dirty  # confirms "Checking for slow types..." with no warnings

# 6. (JSR only) Every entrypoint in exports needs its own @module-tagged doc comment — this is
#    what JSR's package-score "Has module docs in all entrypoints" check requires, separate from
#    doc-lint above. Missing ones show up as real gaps here, not as a doc-lint error.
for f in $(grep -oE '"\./[^"]*":\s*"\./[^"]+\.ts"|"\.":\s*"\./[^"]+\.ts"' deno.jsonc | grep -oE './[^"]+\.ts$'); do
  grep -q '@module' "$f" 2>/dev/null || echo "MISSING @module: $f"
done
echo "(no output above = every entrypoint has one)"
```

To confirm a count (e.g. `doc --lint` errors) isn't a regression and not just "it didn't go up":
`git stash && deno doc --lint <entrypoint> 2>&1 | tail -3 && git stash pop`.

## Phase 5 (OPTIONAL) — Validate against a real production project

**Don't do this by default.** Only if the user explicitly asks, or ask them once: "is there a real
production project using this library that I could check to validate the documentation matches real
usage?" If there isn't one or they don't ask, don't offer this extra step.

If there is a real project (or several):

1. Check which version of the library that consumer has pinned — if it's older than the one you're
   documenting, confirm the changes between versions are additive (don't break what you're about to
   validate) before assuming behavior is the same.
2. **Grep every real import of the package across the ENTIRE consumer repo first**
   (`grep -rln "@scope/pkg" src`), not just the 2-3 files you happen to open — a systemic convention
   (e.g. "every single import uses the root package, never a subpath export") only becomes visible
   at full-repo scale; sampling one or two files risks concluding "mixed usage" from too small a
   sample, or missing the pattern entirely. If the real import style consistently diverges from what
   your docs show as the primary pattern (e.g. docs lead with subpath imports but 100% of real usage
   imports from the root), that's a real finding — the docs' recommended pattern should match what
   actually gets used, with the alternative demoted to a secondary mention, not the reverse.
3. Look specifically for:
   - API option combinations used in production that your docs never show.
   - Real file/module naming conventions vs. the documented ones — and if they differ, find out
     WHETHER it's the library itself that enforces them or a satellite tool (e.g. a separate
     CLI/bootstrapper) — don't assume the library you're documenting is the one imposing the
     convention.
   - Instance/dependency access patterns actually used (do they only use the documented singular
     getter, or is there a dynamic/plural getter for the multiple-dependency case that your docs
     never mentioned?).
   - Middlewares: is the base primitive used directly, or almost always a higher-level decorator
     built on top of it by an internal org package? If it's the latter, your docs should explicitly
     say that's the expected pattern in practice.
   - Error handling, environment variables/constants actually set.
   - The real origin of any concrete class used in an example (does it come from this library, from
     a sibling, or did the app itself write it?) — don't assume it, grep the real import.
   - **Compositional "recipe" patterns**: real code often combines two primitives you've *already*
     documented individually into a non-obvious technique neither one's own doc would ever suggest
     (e.g. masking a search term with the same masking function, specifically to build a
     partial-match query filter against a field that's stored masked). These recipes are exactly
     what isolated API-reference docs never show — when you spot one, add it as its own short
     example near whichever primitive is the more surprising half of the combination.
4. **Strict scope**: the goal is to validate/complete THIS library's docs, not document the
   satellite libraries the real project also uses. If a real pattern belongs to another library,
   note it as "out of scope, lives in `<package>`" and move on — don't write a guide for it here.
5. **If a finding turns out to be a real behavior bug in the library** (not just a doc gap) — for
   example, a decorator that doesn't correctly apply something a sibling decorator does apply —
   **stop and ask the user** before patching the code. Fixing documentation proceeds autonomously;
   changing runtime behavior is the user's call, even if the fix is obvious and one line long.
   Verify the finding rigorously before reporting it (read the full implementation of the code path
   involved, not just the symptom) — a poorly-verified "looks like a bug" costs more tokens than
   confirming it properly once. And don't assume your first conclusion is correct just because it
   sounds reasonable: if while implementing the fix you discover it doesn't solve what you thought
   (e.g. registration works but runtime consumption never reads that registration), say so
   explicitly and reconsider — don't leave a half "fix" in place that gives false confidence.

## Expected report format (to avoid wasting tokens narrating)

```
README: X→Y lines. Real bugs fixed: <short list with line>.
docs/: N guides (new: ..., extended: ...). Coverage: every exported symbol has a documented home,
  except <list with reason, if any>.
CHANGELOG: entry [X.Y.Z] added with <categories>.
Verification: fmt/links/anchors/symbols/check/test/doc-lint — all clean (or documented exceptions:
  ...).

Validation against a real project (if done):
- Confirmed: <what matched the docs>.
- Gaps found and fixed: <file:line in the real project → what was added/fixed in the docs>.
- Possible behavior bug (needs your decision): <description + where the code is>.
```

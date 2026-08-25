---
name: skill-and-agent-authoring
description: How this repo's own skills and agents get created and maintained — tier placement (universal/ecosystem/packages), the real distribution mechanism, when a package earns a builder agent, the validation discipline every new skill/agent goes through, and cross-agent boundary upkeep as the roster grows. Use when creating a new skill or agent, deciding where it belongs, or reviewing whether an existing one still holds up. Does NOT cover the content of any specific skill/agent, or how a session safely DISPATCHES them once built — that adjacent concern is zanix-agent-dispatch-discipline's job.
---

Grounded in this repo's own real authoring history — three ecosystem skills
(`zanix-envvar-conventions`, `zanix-observability-conventions`,
`naming-and-structure-conventions`) and two agents
(`conventions-sweep`, `consumer-conventions-reviewer`) built and validated
the same way, not a theoretical process. Also grounded in the founding
"claude-skills redesign" design artifact where its definitions still
hold — flagged below where reality diverged from that artifact's original
plan, since the artifact itself is a one-time planning document, not
something kept in sync as the system evolved.

## Golden rule (token savings)

- **Never write a skill from a summary of a summary.** Every real skill this
  repo has ground itself in a full audit with real file:line citations
  across the actual repos — not general knowledge, not a remembered
  impression of how something works.
- **Verify every load-bearing claim against real code before writing it
  down** — including claims from a source you trust. An audit artifact that
  was otherwise careful still had a path missing a segment
  (`ConnectorCoreModules`'s real location) and wrongly assumed two symbols
  weren't public exports when they were (`visuallyHiddenStyle`,
  `DLQAggregator`) — both caught by checking, not by trusting the source.
- **A new skill/agent isn't done at "written."** It's done after: symlinked
  in both profiles, referenced from every real consumer, and validated
  against a real scenario. Skipping the last step means shipping something
  nobody's confirmed actually works.

## Tier placement — decide before writing, with a real criterion

- **`universal/`** — general engineering discipline, portable outside this
  ecosystem in principle (test coverage, docs/JSDoc audits, release
  mechanics, naming/casing conventions). The test: could this skill's
  *content* apply to an unrelated codebase with the Zanix-specific examples
  swapped out? `naming-and-structure-conventions` went here, not
  `ecosystem/`, on exactly this reasoning — it's a style/quality-discipline
  skill, the same shape as `documentation-voice`/`complete-test-coverage`,
  not a description of how Zanix's own architecture works.
- **`ecosystem/`** — describes how THIS framework's own architecture works,
  or how this repo's own tooling gets maintained (this skill included) —
  not portable outside this specific ecosystem. `zanix-envvar-conventions`
  and `zanix-observability-conventions` are here because they document real
  precedents specific to how Zanix packages resolve config and log/throw,
  not a generic convention borrowed from elsewhere.
- **`packages/<repo>/`** — distills one specific package's own `docs/`
  (`engineering.md` or equivalent). **Never duplicates that doc — distills
  it.** The package's own docs stay the source of truth; the skill is what
  a Claude session actually needs loaded to work in that package without
  re-reading the whole doc tree every time.

## Which symbols/functions earn a catalog entry — discoverability, not exhaustiveness

A `packages/<repo>/` skill describing a package's real exported surface
(helpers, decorators, utilities) should never try to be a mirror of that
package's own JSDoc/JSR docs — `jsdoc-jsr-audit` already exists to keep
JSDoc itself accurate, and JSR renders it live from real source. Restating
a symbol's full signature/behavior in a skill duplicates that source of
truth and adds a second place it can go stale — confirmed real: two skills
still named `regex.ts`'s pre-rename camelCase constants after the source
itself had already moved to `UPPER_SNAKE_CASE`, exactly the kind of drift
a skill-as-JSDoc-mirror invites.

**The real test for whether a symbol earns a mention**: would a session
working in a DIFFERENT package plausibly not think to check this package
for it, and reinvent it locally instead? Not "does it have JSDoc" (nearly
everything does) — whether its very EXISTENCE is discoverable without
already knowing to look for it. Four real, confirmed cases that failed
this test and had to be fixed after the fact: `confinePath` (a real
path-traversal guard nobody thought to check for before hand-rolling
similar logic), `sanitizeUrl` (an XSS-scheme guard with a real consumer,
undocumented anywhere in the repo), `isPlainObject` (real consumers in two
other packages, zero skill mentions), and `OBJECTID_REGEX`/
`PERMISSION_REGEX` (hand-templated locally in `cli` because nothing
surfaced that `@zanix/utils`'s validator module already had — or should
have had — the equivalent). A routine, self-explanatory helper
(`generateUUID`) doesn't need this treatment — its own one-line JSDoc is
already sufficient once someone's looking at the source at all, since
nothing about it invites cross-package reinvention.

When a symbol does earn a mention, keep the entry to what JSR/JSDoc can't
already tell you at a glance: real cross-repo consumers, a non-obvious
gotcha the signature alone doesn't reveal (e.g. rejecting class instances
via a prototype check, not just `typeof`), or how it relates to a sibling
symbol — not a restatement of the signature's own JSDoc prose.

**A different test applies to a "moved away"/"no longer here" section**
(e.g. `utils-config-and-project-helpers`'s "Moved to `@zanix/cli` — no
longer in this package") — don't apply the discoverability test above to
these, they're answering the opposite question. A symbol catalog entry
prevents REINVENTING something that already exists; a moved-away note
prevents ASSUMING an old location still works. The real risk it guards
against: a session (or a stale skill/test file — the confirmed real case
is `zanix-tree-jsr-fetch.test.ts` still importing the pre-rename
`versionRegex`, filed as a real issue) references the old location,
fails, and burns real time debugging an import that was never going to
resolve. This content doesn't decay the way a behavior catalog does — once
true, "referencing X from here fails" stays true — so keep it as long as
the wrong assumption stays plausible (a real historical location someone
might reasonably still expect), not indefinitely once the migration is old
enough that nobody would guess the symbol was ever here.

## How skills/agents actually get distributed — corrected from the original plan

The founding artifact's original design called for a per-repo
`scripts/link-skills.sh` symlinking only the relevant tier subset into each
Zanix repo's own `<repo>/.claude/skills/`. **That script was never built.**
Every real Zanix repo's own `.claude/` directory holds only Claude Code's
own local settings (`settings.local.json`, lock files) — confirmed via a
direct `find`, no skills/agents content in any of them.

What's actually in place: a **user-profile-level** symlink into
`~/.claude/skills/`+`~/.claude/agents/` and
`~/.claude-personal/skills/`+`~/.claude-personal/agents/` (two separate
`CLAUDE_CONFIG_DIR` profiles) — every skill and agent visible from any
repo, any time, regardless of which project you're in. **A new skill or
agent needs a real symlink in BOTH profiles, in the same turn it's
created** — missing the second profile is a real, already-happened mistake
(a historical backlog of 78 missing skill symlinks in one profile was found
and fixed, confirmed via `readlink`) — verify with `readlink` on
both paths, don't assume the first one succeeding means the second did too.

## Creating a new skill

1. **Ground it in a real audit**, not memory — a dispatched research pass
   across the real repos with file:line citations, or an existing artifact
   you verify rather than transcribe.
2. **Decide the tier** against the criterion above before writing a word.
3. **Cross-check every load-bearing claim directly** against the real repo
   the skill describes — don't trust the audit's own citations blind.
   **Before writing a rule's own WHY, grep sibling skills for whether that
   rationale already lives somewhere** — reference it, don't restate it.
   This is what would have prevented an entire audit's worth of real
   duplication findings: the same sentence (a `_ENV`-constant rationale, a
   CHANGELOG-micro-edit rationale, `documentation-voice`'s 4-part rule
   restated 3 separate times, a test-file-size ceiling's own reasoning)
   independently written into 2+ files instead of one owning it and the
   rest pointing at it. The tell is writing more than "see `X`" for
   something `X` already explains in full — if you're about to justify a
   rule, not just state it, check whether the justification already has a
   home first.
4. **Match house style**: read 1-2 sibling skills in the target tier first.
   Golden Rule section → numbered rules or topic sections → resolved edge
   cases inline (don't leave a genuine ambiguity unresolved if the audit
   already settled it) → a tracking section for real, still-open gaps → a
   checklist → an explicit "Out of scope" section.
5. **Symlink both profiles**, same turn, verified.
6. **Reference it from every real consumer** — each relevant agent's own
   "Skills to load" section, with a bullet *tailored to that package's own
   real precedent*, never a generic copy pasted across agents.
7. **Validate against a real scenario** — ideally a dispatched agent with NO
   access to the authoring session's own context, given only the skill and
   real code, asked to reach a verdict independently. This has surfaced a
   real gap every time it's been tried: a CSS-in-JS style-object casing
   edge case the rules didn't cover, a boot-time-error exemption an
   observability rule assumed away. Treat "the validation came back clean"
   as a real possible outcome, not evidence the step is unnecessary — it
   only stays cheap because it's actually run, not skipped on the
   assumption a careful author already got it right.
8. **Fix whatever the validation finds immediately** — don't ship a known
   gap in a skill that's about to be relied on.

## Creating a new agent

1. **Confirm the real two-condition criterion**, don't build on "would be
   useful": (a) a genuinely repeatable workflow exists — not a static
   architectural rule to remember, an artifact added the same way multiple
   times; (b) enough real skills already exist for the agent to have
   something concrete to orchestrate. Several repos (`asyncmq`, `space`,
   `app`, `core`, `utils`) correctly have no builder agent because they
   fail condition (a) — that's a confirmed conclusion, not a gap to revisit
   without a genuinely new, repeatable pattern surfacing.
2. **Model it on the closest real sibling**, don't invent a new shape — a
   new sweep agent should read an existing sweep agent's file first (its
   Golden Rule, Output format, Out of scope shape); a new reviewer agent
   should read an existing reviewer agent's file the same way. **The real,
   observed heading sequence every agent in this repo already follows —
   spelled out here because nothing named it explicitly before, and its
   absence is exactly what let 5 of one audit's 11 real consistency findings
   happen (2 agents missing `## Skills to load` entirely, 3 with no
   `## Definition of done` heading — the content existed, buried inline,
   just never under its own heading):** frontmatter → intro paragraph →
   `## Golden rule (token savings)` → `## Skills to load` → the concrete
   workflow (numbered steps or topic sections, whatever the task shape
   needs) → `## Definition of done` (whenever completion is gated by
   something beyond "the workflow steps finished," e.g. a
   `feature-completeness-conventions` check) → `## Docs move in the same
   change` (only when the package's own docs need updating as part of this
   agent's workflow) → `## Out of scope — do not do these`. Deviate only
   with a real reason, not because a section felt optional.
3. **Draw explicit boundaries against sibling agents in "Out of scope"** —
   name the specific look-alike task and which agent actually owns it.
4. **Two standing conventions apply to every agent's Golden Rule, not just
   the ones that happen to have copied them so far — write the same
   sentence once here, don't re-derive fresh prose for it in every new
   agent file:**
   - **Report once, at the end.** Pass/fail per check, files added/edited,
     one line per caution/gotcha flagged — never a running narrative of
     every file read along the way, and never a second full summary after
     an already-compact report.
   - **Verify structural claims empirically, not by trusting a source.**
     A grep or a small real test against actual current code, not general
     knowledge or a remembered impression — even when the claim comes from
     a trusted audit or a sibling skill's own prose. This is what "Verify
     every load-bearing claim against real code" (Golden Rule, above) means
     applied specifically to an agent's own runtime checks, not just to
     this skill's own authoring process.
5. **Symlink both profiles**, same turn, verified.
6. **Validate against a real scenario** — a real sweep run across real
   repos, or (when no real target exists yet) a deliberately-crafted
   fixture with planted issues, built specifically to test the agent
   without touching anything real. Both shapes have found real gaps in a
   brand-new agent's own instructions on first use.
7. **Update `README.md`/`CATALOG.md`** — and audit the *whole file*, not
   just the table you're touching, for other staleness while you're in it,
   rather than only appending the new row. `README.md`'s own agent table was found
   missing six unrelated agents the same day two new ones were being added
   — the gap wasn't caused by the addition, but touching the table is
   exactly when it gets noticed and is cheapest to fix. **"Whole file," not
   "whole table," is deliberate**: a real completeness finding lived in
   narrative roadmap prose OUTSIDE either table (both files kept saying
   `utils` "doesn't need a builder" long after `utils-builder` (then named
   `utils-linter-builder`) shipped and had its own correct table row) — a check scoped only to the
   table row would have missed it, since the row was already right. **Each
   doc has one job — don't let the
   new row blur it.** `README.md` is the one-line "what does this do"
   reference; `CATALOG.md` adds only the audience dimension and the
   file/directory layout, explicitly per its own intro. Confirmed real:
   `CATALOG.md`'s Agents table once carried a "Purpose" column restating
   README's own description near-verbatim — added without noticing it
   contradicted the file's own stated design. A new row for either file
   states only what that file actually owns; if you find yourself writing
   the same sentence into both, one of them doesn't need it.

## Cross-agent boundaries need active upkeep, not just a one-time decision

Every new review/sweep agent can silently break a sibling's own boundary
logic. Confirmed real: `consumer-conventions-reviewer` was written to
redirect to `architecture-reviewer` when pointed at a library repo by
mistake — but `architecture-reviewer` itself had no idea its new sibling
existed, so the redirect only worked one direction. Whenever a new
review-shaped agent is added, check whether any EXISTING agent's own scope
language needs a cross-reference added, not just whether the new agent's
own scope is internally correct.

**This check has two distinct shapes — checking only one of them is why the
same bug reappeared 4 more times after the first fix above.** (1) A direct
"redirect to X"/"distinct from X" pointer that needs its OWN reciprocal
pointer added to X — the shape already described above. (2) **A sibling
ENUMERATION list** — an agent that names "its" set of periodic-sweep peers,
or every other reviewer, as a flat list (`benchmark-sweep`'s "any other
periodic-sweep axis: ecosystem-maintenance, dependency-drift,
dependency-direction-sweep, dead-code-sweep, documentation-agent" is the
real example that went stale) — where a brand-new sibling needs to be
ADDED to every existing such list across the repo, not just get its own
list right. When creating a new sweep/review agent, grep every existing
agent for the CATEGORY of work the new one does (not just its exact name,
which obviously won't be found) to find these enumeration lists before
considering the boundary work done.

## When a skill outgrows itself

Watch for one skill file accumulating multiple genuinely separate
responsibilities over time (the historical `zanix-libraries-architecture`
case: six-plus mixed concerns in one 800+-line file). Split when found —
into the tier each split piece actually belongs in, per the placement
criterion above. **Trim, don't delete wholesale** — a skill split or
corrected almost always still contains real, verified knowledge; only the
genuinely obsolete or misplaced section (a stale bug description, a dated
migration note) gets removed, not the surrounding accurate content.

**A split or rename isn't done until every inbound citation is fixed too.**
`grep -rl '<old-skill-name>' .claude/` across the whole repo before calling
either done — a skill name cited by another skill's prose (a "for X, see
`old-name`" pointer) doesn't get automatically corrected just because the
content moved. Confirmed real: `app-manifest-and-composition` cited
`app-distributed-runtime`/`app-platform-features` in 5 places, long after
both were split into 5 real, differently-named skills — nobody grepped for
the old names when the split happened, so the pointers just went stale
silently until an unrelated audit caught them.

Also don't over-create: a cross-cutting discipline that fits in a couple of
sentences doesn't need its own skill file. "Verify structural claims
empirically" was drafted as a candidate standalone skill once and
deliberately retired in favor of a short instruction embedded directly in
the agents that needed it — no longer just two. Confirmed current, by
grep, not by memory: `architecture-reviewer`, `consumer-conventions-reviewer`,
`datamaster-builder`, and `zanix-feature-builder` all carry their own
empirical-verification instruction today, and `dependency-drift`'s entire
job description IS this discipline (a whole agent whose only output is
"does the real, current code match the claim," not a skill that needed
separate embedding). The count keeps growing as new agents are built —
don't re-cite a stale number here; re-grep for `empirical` across
`.claude/agents/` before quoting one. A new file still isn't automatically
the right container just because a concern is real and now repeated
several times — see "Creating a new agent" below for where this and
"report once" now get codified once instead of drifting further.

## Transient/in-progress state doesn't belong inside a skill's own prose

A skill describes the stable, current standard — not an in-flight
migration or a known-but-not-yet-fixed violation, sitting in the skill file
indefinitely. Where a real gap exists and isn't fixed in the same change,
prefer filing it through `zanix-issue-reporting` (once built) over
accumulating it as a growing "Known current gaps" section — that pattern
already produced skill files carrying open-ended backlogs of real findings
with no durable owner beyond "whoever reads this skill next." A short,
temporary tracking note is fine while a fix is actively in flight; it's not
a permanent home for it.

## Checklist before calling a new skill/agent done

- [ ] Grounded in a real, verified audit — not memory or an untranslated
      artifact?
- [ ] Tier decided against the real criterion, not by feel?
- [ ] Every load-bearing claim checked against the real repo it describes?
- [ ] House style matched against a real sibling file — (agents only) the
      real heading sequence specifically: Golden rule → Skills to load →
      workflow → Definition of done (if applicable) → Docs (if applicable)
      → Out of scope, not just "read a sibling and vibe-match it"?
- [ ] Any rule's own rationale (not just the rule itself) checked against
      sibling skills first — referenced instead of restated if it already
      has a home?
- [ ] Symlinked in both `~/.claude/` and `~/.claude-personal/` profiles,
      verified with `readlink`?
- [ ] Referenced from every real consumer, each with a tailored bullet?
- [ ] Validated against a real scenario, gap found (if any) already fixed?
- [ ] (Agents only) Explicit boundary check against every sibling agent's
      own "Out of scope" AND every sibling-enumeration list across the
      repo (grep by category of work, not just by the new agent's name) —
      not just the new agent's own scope language?
- [ ] (Split/rename only) `grep -rl '<old-name>' .claude/` run, every
      inbound citation fixed, not just the split/rename itself?
- [ ] `README.md`/`CATALOG.md` updated, and the surrounding WHOLE FILE
      (not just the table) audited for unrelated staleness while touching
      it?
- [ ] Each new row states only what that file actually owns (purpose in
      README, audience/layout in CATALOG) — no sentence written into both?

## Out of scope — do not do these

- The content of any specific skill or agent — this only covers how they
  get authored/maintained, not what any one of them should say.
- How a session actually dispatches these agents safely (the standing
  "named agent must run the work" gate, concurrent-dispatch verification,
  cross-session coordination) — that's `zanix-agent-dispatch-discipline`'s
  job, a using-time concern distinct from this skill's authoring-time one.
- Deciding whether a package's real workflow is "repeatable enough" without
  checking it against real, multiple, genuinely-identical-shaped instances
  first — that's the same discipline `zanix-envvar-conventions` already
  applies to env-var patterns, not a new judgment call to invent here.
- Reviving the original per-repo `scripts/link-skills.sh` distribution
  model without a concrete reason the current profile-level mechanism has
  actually failed — it hasn't, as of this writing.

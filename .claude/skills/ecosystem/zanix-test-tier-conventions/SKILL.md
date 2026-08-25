---
name: zanix-test-tier-conventions
description: Which of unit/integration/functional a new test belongs in — the real ecosystem-wide default every zanix new-scaffolded project gets (sourced from @zanix/utils's own real example.test.ts templates), and how a repo's own more specific testing-architecture skill (like @zanix/cli's) refines it without replacing it. Use whenever adding a test to any Zanix repo — library or consumer — and deciding which @tests/ subfolder it belongs in.
---

The single source of truth for tier PLACEMENT across every Zanix repo —
library and consumer alike. Confirmed real, not assumed: every repo under
`~/Documents/Development/ZanixLibraries/` (`datamaster`, `server`,
`notifications`, `auth`, `admin`, `space-ui`, `asyncmq`, `core`, `app`,
`space`, `utils`, `cli`) has a real `src/@tests/{unit,integration,functional}/`
structure — this isn't a `cli`-specific or consumer-only convention, it's
the ecosystem default. Doesn't cover test CONTENT/coverage completeness —
that's `complete-test-coverage`'s job, deliberately stack-agnostic and not
duplicated here.

## Golden rule (token savings)

- **Don't duplicate these definitions into your own agent/skill file.**
  Reference this skill instead — the definitions live in exactly one place;
  match `skill-and-agent-authoring`'s own rule about not writing the same
  sentence into two files.
- A repo's own more specific testing skill (`cli-generator-testing` for
  `@zanix/cli`) REFINES this default with real repo-specific detail — it
  doesn't replace or contradict the three tier names/shapes below. Load
  both when one exists; this skill alone when it doesn't.

## The real source

`zanix new` scaffolds `@tests/{unit,integration,functional}/example.test.ts`
into every new project, confirmed via
`cli/src/commands/new/lib/tree/projects/commons.ts` (`jsr: '@zanix/utils'`
per tier). The real descriptive content lives in `@zanix/utils`'s own
`src/templates/src/@tests/{unit,integration,functional}/example.test.ts` —
read those directly if this summary looks stale; they're the actual
published templates, not paraphrased here from memory.

## The three tiers

| Tier | Real definition | 
| --- | --- |
| `unit/` | A single function/small unit tested in isolation — no external systems, fast, self-contained. |
| `integration/` | Multiple parts of the system interacting — may involve real or mocked external services (APIs, databases). The interaction itself is what's under test, not one component's internals. |
| `functional/` | The whole feature end-to-end, from the user's/caller's perspective — a real-world scenario, often crossing multiple components and external systems. |

## Applying this to a new artifact/feature

Most single-unit work (one handler/interactor/repository/connector/rule,
tested with its own dependencies mocked) belongs in `unit/`. Reach for
`integration/` when the test genuinely exercises more than one real piece
wired together — including, confirmed as a real recurring shape across
`cli`'s own top-level commands (`build`/`new`/`prepare`), a **"is this
artifact correctly wired into the real system" check** (a fresh instance,
call the real registration function, assert on the real resulting
shape/options) — this is NOT a unit test of internal logic, it's proof the
piece is actually connected, and it belongs in `integration/` even when
every individual assertion looks unit-test-sized. `functional/` is for a
real end-to-end path (a real subprocess, a real request through the running
app) — reach for it only when `unit/`+`integration/` genuinely can't prove
the feature works as a whole.

**Fixed, real worked example of the "don't default to `unit/` by habit"
case**: `@zanix/cli`'s new `report-issue` top-level command originally
landed every test in `unit/`, with a "command should be correctly defined"
style assertion duplicated inside a unit test file instead of added to
`integration/commands.test.ts` alongside `build`/`new`/`prepare`'s own real
entries there. Since fixed — `report-issue` now has its own real entry in
`integration/commands.test.ts`, and the duplicate unit-tier assertion was
removed. Kept here as the precedent for what the fix looks like, not as an
open violation. Also confirmed: a
mocked unit test paired with a real, live companion test where safely
possible is itself a real, existing pattern (`commands/new/lib/tree/
info.test.ts`'s mocked unit test + `functional/zanix-tree-jsr-fetch.test.ts`'s
real, unmocked JSR-registry call) — `report-issue`'s own tests cited this
exact precedent but only followed half of it (mocked `fetch` throughout,
no live companion test), reasonable here specifically because the real
action is a *mutating* third-party side effect (filing a real, spammy
GitHub issue) rather than a safe read — but that tradeoff needs to be
written down as a deliberate exception, not left silent, if the same
`safe-to-mock, unsafe-to-run-live` category comes up again.

## Three recurring misplacement patterns — confirmed real across a 12-repo audit

A full test-tier placement audit across all 12 Zanix repos (2026-08-21,
every file opened and read, not just named) found real, repeated
misplacements — not one-off mistakes. Three shapes account for nearly
everything found; watch for these specifically, not just "does this feel
unit-sized":

- **Pattern A — a real wiring/registration call with no mock, filed under
  `unit/`.** The `report-issue` example above generalizes: any test that
  calls a REAL registration function (`register<X>`) and asserts against
  the REAL resulting registry/container state, with nothing mocked, is
  proving the piece is wired into the real system — that's `integration/`'s
  job by definition, even when every individual assertion looks
  unit-test-sized. Confirmed in this shape, independently, in 5 of the 12
  repos (`auth`, `asyncmq` ×2, `admin`, `datamaster`, `core`) — the single
  most common misplacement found.
- **Pattern B — a real server/socket/network dependency inside `unit/`.**
  The sharpest of the three: `unit/`'s own definition is "no external
  systems" — a test that calls `bootstrapServers()`/`.serve()` or an
  equivalent, binds a real port, **or** makes a real `fetch()` against
  `localhost`, is not isolated by construction, regardless of how contained
  it looks. **Any ONE of those three conditions alone already disqualifies
  it — binding a real port with no accompanying `fetch()` still counts.**
  Confirmed real, not hypothetical: this exact ambiguity (an earlier,
  self-contradictory draft of this bullet used "and" in the definition
  sentence but "no exception" in its own closing line) caused a real
  undercount — a fix pass read the conjunctive phrasing and left 4
  port-binding tests in `server/unit/webserver/mod.test.ts` untouched
  because they didn't also call `fetch()`, only caught by a later
  independent `test-tier-sweep` run using the correct disjunctive rule.
  This is a real risk, not just an organizational nit — hardcoded ports
  inside a tier that's supposed to run fast and parallel-safe is a genuine
  flakiness/collision risk. Confirmed in 4 of the 12 repos (`server`,
  `app` ×2, `space`), each with its own hardcoded port — re-verify this
  count before citing it, more have since surfaced in some repos.
  **An ephemeral port (`port: 0`) still counts, don't downgrade it.**
  Confirmed real, since fixed: `datamaster`'s
  `s3-object-storage-error-classification.test.ts` and
  `unit/observability/connector.test.ts` used to bind a real
  `Deno.serve({port: 0})` listener inside `unit/` — the specific
  collision-flakiness risk the rationale above describes was reduced (no
  fixed port to collide on), but the test still wasn't isolated by
  construction: a real listening server and a real network round-trip
  inside `unit/`, which the rule's own closing sentence already covers
  regardless of port strategy. Both have since moved out of `unit/`: the
  whole error-classification file now lives at
  `integration/storage/s3-object-storage-error-classification.test.ts`,
  and the specific listener-using case in `connector.test.ts` was split
  out into `integration/observability/connector-basic-auth.test.ts` (the
  `unit/` file itself stays, minus that one test). Kept here as the
  precedent for what the fix looks like, not as an open violation. Don't
  let "the collision risk is mitigated" read as "so this one's fine" —
  the rationale explains why hardcoded ports are the worst case, it isn't
  an exhaustive list of what qualifies. If a test binds a real network
  listener, it belongs in `integration/` or `functional/`, never `unit/`
  — no exception, fixed or ephemeral port alike.
- **Pattern C — the mirror mistake: a pure, isolated function or class
  filed under `integration/` out of habit or file-proximity**, with no
  actual interaction between real parts. Confirmed in 4 of the 12 repos
  (`datamaster` ×2, `notifications` ×2, `utils`, `server`) — often sitting
  right next to a correctly-placed sibling that DOES test real interaction,
  making the contrast easy to spot once looked for. A test of one pure
  function/class in isolation is `unit/` regardless of which directory
  habit or proximity to a related integration test suggests. **Don't judge
  by the file's name** — `server`'s own
  `integration/program/{base,context,decorators,middlewares}.test.ts` sound
  like infrastructure/wiring tests, but each one instantiates a single
  metadata-container class (`RegistryContainer`/`ContextContainer`/
  `DecoratorsContainer`/`MiddlewaresContainer`) directly and asserts only
  against its own public API, zero interaction with any other real module —
  the exact same class family's own `unit/program/registry.test.ts` tests
  correctly, in `unit/`, making the contrast direct. A plausible-sounding
  name (`decorators.test.ts`, `middlewares.test.ts`) is not evidence of real
  cross-module interaction — read the actual test body.

**A known, real contradiction in `@zanix/utils`'s own published
templates**: the same audit found `src/templates/src/@tests/
{integration,functional}/example.test.ts` — the actual code `zanix new`
scaffolds into every new project — has example CODE that contradicts its
own prose (the `integration/` example is actually an isolated pure
function; the `functional/` example is a stub touching no real system).
The prose in those templates still matches this skill's own table above;
the CODE doesn't. Trust this skill's table over the template's example
code specifically, until that contradiction is fixed at the source (a real
`@zanix/utils` fix, not a skill fix — tracked separately, not solved here).

## Checklist before calling a new test's placement done

- [ ] Does the test's own shape match the tier's real definition above —
      not just "it's new, so `unit/`"?
- [ ] If the artifact is a new top-level command/entrypoint (not a
      variation on an existing generator/pattern), does it have an
      `integration/`-tier "correctly wired/defined" check, matching the
      real `build`/`new`/`prepare` precedent — not just internal-logic unit
      tests?
- [ ] If a real external call can't safely run in a test (a mutating
      side effect, not a safe read), is that documented as a deliberate
      choice, not silently absent?
- [ ] Does the repo have its own more specific testing skill to layer on
      top of this default (check before assuming this skill alone is the
      complete picture for that repo)?
- [ ] Does the test call a real registration/wiring function with nothing
      mocked (Pattern A — belongs in `integration/`, not `unit/`, however
      small each assertion looks)?
- [ ] Does the test bind a real server/socket or make a real network call
      (Pattern B — never `unit/`, no exception, this is a real flakiness
      risk not just an organizational one)?
- [ ] Is the test actually one pure, isolated function/class with no real
      interaction, just sitting in `integration/` out of habit or file
      proximity (Pattern C — belongs in `unit/`)?

## Out of scope — do not do these

- Test coverage completeness/what scenarios to cover — `complete-test-coverage`'s
  job entirely, deliberately not duplicated here.
- A repo's own more specific testing-architecture rules (exact paths,
  per-artifact conventions) — that's the repo's own skill's job when one
  exists; this skill only owns the shared, ecosystem-wide default.

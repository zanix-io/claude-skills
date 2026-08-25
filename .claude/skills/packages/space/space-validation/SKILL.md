---
name: space-validation
description: The build-time/dev-time document validation system — the three independent axes (severity, opt-in, strict), the basis field that decides what's normative, the five real error-severity rules, and what's deliberately never checked (runtime facts, human judgement) rather than guessed. Use when a validation finding needs triage, or when configuring/exempting a route from it.
---

This is a document validation system, not an SEO checker — HTML conformance,
accessibility, search presentation, social metadata, and PWA installability
are treated as separate concerns with separate authorities, never folded
together into one "best practice" requirement. File:line references point
at `~/Documents/Development/ZanixLibraries/space` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- Triage a finding by its `basis` field first (normative vs. not) — that
  alone tells you whether it can ever legitimately be an error, before
  looking at anything else about the rule.
- Trust "not checked, and why" in a validation report as a real, distinct
  category from "checked and clean" — don't re-verify something the report
  already says it couldn't check.

## Running it

Runs automatically during `zanix space build` and `znx space dev`. CLI
flags (`--validation`, `--no-validation`, `--validation-strict`,
`--validation-category`) are documented in `@zanix/cli`'s own `zanix space`
guide.

## The three independent axes

| Axis | Question |
| --- | --- |
| `severity` (`info`/`warning`/`error`) | How bad is this finding? |
| `optIn` | Is this rule active by default? |
| `strict` (project-wide) | Should active warnings be treated as errors? |

Effective severity resolves in one fixed order: catalog severity → per-rule
override → strict promotion → effective severity. `strict` means no active
warning stays a warning, including one a project set explicitly — but it
never promotes `info`, since strict enforcing an `info`-level signal would
mean the framework enforcing something it itself says isn't a real
requirement.

## `basis`: what a rule actually rests on

| Basis | Normative | Meaning |
| --- | --- | --- |
| `spec` | yes | HTML Standard or other protocol requirement |
| `accessibility` | yes | WCAG success criterion, or a W3C ACT rule mapped to one |
| `installability` | yes | Documented PWA installability criterion |
| `search-engine-recommendation` | no | Documented search-engine guidance |
| `ecosystem-recommendation` | no | Widely-held practice with real rationale |
| `framework-invariant` | no | A rule `@zanix/space` imposes on its own output |
| `project-convention` | no | A convention this framework's scaffolding follows |
| `heuristic` | no | A signal with no primary source at all |

A normative rule always cites its reference, so a build-failing finding can
always answer "says who." Only a normative basis can ever be error-severity
— a rule qualifies as an error only if it's detected deterministically,
represents an invalid document or an unambiguous contradiction, has
near-zero false positives, and has a clear "why should this block the
build" answer. Recommendations, heuristics, and content-or-judgement-dependent
rules are always warnings or info, however strongly held.

## The five real error-severity rules

| Rule | Basis | Why it blocks |
| --- | --- | --- |
| `A11Y002` | accessibility | Viewport prevents zoom (WCAG 1.4.4 AA, via ACT rule b4f0c3) |
| `DOC003` | spec | The response isn't a document at all |
| `PWA001` | installability | Icons omit 192 or 512 — the app can't be installed |
| `FW001` | framework-invariant | Head resolution produced conflicting canonical URLs |
| `FW003` | framework-invariant | Resolved head didn't reach the rendered document |

## `<h1>` is not a requirement — a corrected misconception

A missing `<h1>` is a `warning`, basis `project-convention` — never an
error. It's not an HTML Standard requirement, not a WCAG success criterion,
and Google Search Central documents no requirement about heading counts.
It's reported because zero `<h1>` reliably signals an incomplete template,
not because it's mandated anywhere. Heading order and multiple `<h1>`s get
the same treatment: both `info`, both off by default. Opting into `strict`
enforces them; nothing enforces them by default.

## The two phases

| Phase | Decidable from | Runs |
| --- | --- | --- |
| `static` | Modules, layouts, routes, configuration | Always |
| `render` | Real rendered HTML, real data | `--validation=render`, dev only |

The `render` phase renders each route with no dynamic segments and validates
by comparing extracted semantics, not raw HTML strings — so a rule is
written once and holds for both React and Preact.

## What's never checked, and never guessed

- **Runtime facts** (need a live deployment): a canonical URL actually
  resolving, `og:image` existing at usable dimensions, hreflang reciprocity
  across domains, title uniqueness across dynamic routes, a real 404 vs. a
  soft 404.
- **Human judgement**: descriptive headings/labels, alt text actually
  describing its image, link text conveying its destination, content
  substantiveness.

Neither is approximated — "a confident wrong answer is worse than an
admitted gap." A page whose `head` is a function of loader data is never
called with invented data to check it; head-content rules skip such a page,
and the skip itself is reported, not silent. Every run reports what it
could not check and why, specifically to keep "clean" (nothing wrong) from
being confused with "not checked" (skipped work).

## Configuring and exempting

```ts
export default defineSpaceApp({
  name: 'storefront',
  validation: {
    strict: true,
    rules: { SEO002: true, A11Y007: 'info', SEO001: false },
    exempt: ['internal/**'],
  },
})
```

`rules[id]`: `true` activates an opt-in rule at its catalog severity; an
explicit severity string activates and sets it; `false` turns it off. Some
rules are non-configurable by design and ignore overrides. `exempt`
excludes route patterns from document rules (`*` within a segment, `**`
across segments). **`validation: false` disables it entirely for the
project — no command-line flag re-enables it.**

There's no per-page opt-out today: every `GET` yields a document, redirect,
or `304`, so even an action-only route serves a real document and still
wants a title. Two real exemptions exist instead, neither requiring
anything declared on the page: an unconditional `redirect` (inferred during
discovery, since the page never renders), and a project-level route
exemption via `exempt`.

## Checklist before adding/configuring a validation rule or exemption

- [ ] Does a proposed new error-severity rule genuinely have a `spec`/
      `accessibility`/`installability` basis, deterministic detection, and
      near-zero false positives — or does it belong as a warning/info
      instead?
- [ ] Is `exempt` used for a route-level policy decision, rather than
      reaching for a nonexistent per-page opt-out?
- [ ] If `validation: false` is set, is that genuinely intended for the
      whole project — there's no CLI override to re-enable it selectively?
- [ ] For a page whose `head` depends on loader data, is the validation
      report's "not checked" line for that page actually read, not assumed
      to mean "clean"?

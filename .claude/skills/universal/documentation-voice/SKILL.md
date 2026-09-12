---
name: documentation-voice
description: English-only, present-tense voice for everything published or read by another engineer — JSDoc, README/docs/CHANGELOG, code identifiers, and inline comments — with one real exception for genuine @deprecated markers. Never a reference to an authoring session, a plan, or a phase/stage of work. Also owns comment conciseness: a rationale stated once and linked to, never restated at every site that shares it, and a comment rewritten on change rather than extended with a new caveat next to the old one. Use whenever writing or auditing JSDoc, README/docs, or code in any Zanix package.
---

One voice rule, three surfaces: JSDoc (`jsdoc-jsr-audit`), README/`docs/`/
CHANGELOG (`docs-readme-audit`), and code itself — identifiers, inline
(non-JSDoc) comments (`feature-completeness-conventions`'s per-change gate).
Each sibling skill owns its own mechanics; this is the shared rule they all
apply, kept in one place so it can't drift between them.

## Golden rule (token savings)

- Check this during the same pass as the sibling skill's own review — don't
  make it a separate audit. A violation is caught the same way any other
  finding is: `file:line`, fixed in place.
- Scope to what the diff actually touches (same scoping every sibling skill
  already applies) — not a full-repo language sweep for a small change.

## The rule

- **English, always** — never Spanish or any other language, regardless of
  what language the authoring session happened in. Every one of these
  surfaces is read by an engineer (or a JSR consumer) who never saw the
  conversation that produced it.
- **Present tense, describing what the code/API does today** — never a
  reference to an internal session, a plan, a phase/stage of work, or a past
  that no longer matters to the reader. Not: "this was refactored to...",
  "as part of the migration...", "previously this threw, now it...", "this
  used to silently swallow the error, now it rethrows...", "fixed to also
  validate...", "this bug is now resolved". A comment/doc describes current
  behavior, not its authoring history.
- **Pointing elsewhere doesn't launder the violation.** "See the CHANGELOG
  entry for this fix", "see the PR/issue for why", "resolved in v3.2",
  "see this fix" — these still frame the comment around an authoring EVENT
  instead of the code's own current behavior, exactly like inline narration
  does. The violation is the framing, not whether the narrative sits in this
  comment or one hop away in another document. State the real mechanism or
  reason directly instead of pointing at where it was explained once.
- **Never cite an internal Claude Code skill by name in a real package's shipped source** — a
  skill like `deno-lazy-dependency-pattern` is authoring tooling for THIS workflow; it isn't
  published, isn't part of the package, and a JSR consumer or engineer reading the real code has
  no way to open it. This is worse than the CHANGELOG/PR-laundering case above (that at least
  points somewhere the reader CAN reach) — it points at something that doesn't exist for them at
  all. Confirmed real, not hypothetical: `(see the deno-lazy-dependency-pattern skill)` shipped
  into 18 real source files across 7 Zanix package repos before being caught. State the actual
  mechanism/reason directly in the comment instead (e.g. describe what `nodeModulesDir: "auto"`
  does, don't cite the skill that explains it) — the same "state the real reason, don't point
  elsewhere" fix as the CHANGELOG case, just for a strictly worse pointer.
- **Never name a real consumer/client app by name in shipped documentation** — a specific
  customer's own package or repo (`profiles`, `web`, or any other org-specific
  name that isn't a published `@zanix/*` package) has no business appearing in a PUBLISHED
  library's `CHANGELOG.md`/JSDoc/README/inline comment. It leaks that client's identity and
  internal structure to every JSR consumer or engineer who reads the doc, and — same "points at
  something the reader can't reach" problem as the internal-skill case above — is usually
  meaningless to a reader who has no relationship to that client anyway. A legitimate published
  `@zanix/*` package (e.g. `@zanix/iam`, sometimes shortened to `zanix/iam` in prose) is NOT this
  violation — it's a real, citable ecosystem package, not a client's private repo; the test is
  whether the name resolves to something the reader could actually go find, not whether it's
  formatted with an `@`. Confirmed real: `server/CHANGELOG.md`'s `4.3.0` and `4.2.7` entries,
  `auth/CHANGELOG.md`'s `1.4.0` entry, `iam/CHANGELOG.md`'s entry, `space-ui/CHANGELOG.md`'s
  `Textarea`/`DatePicker.icon` entries, plus inline comments in `auth`'s own test suite and
  `iam/src/space/middleware.ts`, all named `profiles`/`web` by name; separately,
  `iam`'s own default message catalog (`src/space/messages/en/index.json`) had a client's brand
  name hardcoded into a generic, reusable logout string. All caught and fixed
  2026-09-11. Describe the failure generically instead — "a consumer app's own
  `imagePublicOrAuthGuard`...", "two separate consumer
  apps' own cookie-consent guards..." — the technical shape of the bug is what matters and survives
  intact; the client's name never needs to appear at all.
- **Genuine engineering rationale is not session narrative, and stays.** "Two
  decoders because cliffy's `.option()` narrows per call" is a real, still-true
  reason worth keeping — the distinction is between a fact that explains
  *why the code is shaped this way* (keep it) and a note about *when/how it
  got edited* (cut it).
- **State a rationale once, then point — restating it doesn't count as
  linking.** A comment that says "same reasoning as `X` above" or "see `Y`'s
  own doc for the full reasoning" and then re-explains that reasoning anyway
  has paid for both copies and saved nothing. Real, confirmed cases:
  `notifications/src/modules/whatsapp/defs.ts`'s `TWILIO_WHATSAPP_FROM_ENV`
  doc restates in full the sender-number-split rationale `sms/defs.ts`'s
  `TWILIO_FROM_NUMBER_ENV` doc already states, instead of one owning it and
  the other linking; `server/src/typings/targets.ts` pastes the identical
  ~80-word `retryInterval` sentence into three separate JSDoc blocks in the
  same file. When a fact is shared by more than one symbol, exactly one of
  them states it — every other one links to it (`{@link X}`, "see `X`") and
  stops there, no re-summary riding along with the link.
- **A comment gets rewritten when it changes, not extended.** Editing a
  comment because the behavior it describes changed means replacing what's
  there with the current single statement of that fact — not appending a new
  sentence, caveat, or headed sub-section next to what was already there.
  A single JSDoc block that accretes multiple sections each restating the
  same point in new words is this same violation spread across one comment
  instead of several — confirmed real in `auth/src/modules/middlewares/
  page-session.guard.ts`'s doc block, where a "Rotation cadence" section
  makes its one point three times in slightly different words and the
  `@param` doc for that same option restates the section's own conclusion
  again on top of that. If a comment needs a second pass, cut whatever the
  new sentence supersedes — don't leave both standing.
- **`CHANGELOG.md`'s own narrative voice is not this violation — and it is
  scoped to that literal file only.** A changelog entry's whole job is to
  say what changed ("Fixed: X now does Y instead of Z") — that's the
  expected format there, not session narrative leaking into documentation.
  This does NOT extend to a comment inside a `.ts`/`.tsx`/any source file,
  even one that itself tracks historical data (a benchmark baseline log, a
  migration script, a version-tracking table) or is shaped like a changelog
  — those stay held to present-tense-only, same as any other code comment,
  with the one narrow exception already covered below: a genuine
  `@deprecated` JSDoc tag. That exception is about a symbol's real
  deprecation status, not a general license to narrate history in code —
  don't read it as overlapping with this point. The test: is this literally
  `CHANGELOG.md` (or a `## Changelog` section in a README), not merely
  changelog-*shaped*? A real, confirmed case of this
  exact confusion: a benchmark suite's own `baseline.ts` recording a
  scenario's re-measurement with "re-recorded... it previously never
  exercised..." — that file is source code with a comment, not the
  CHANGELOG, and the comment should describe what the row measures now, not
  its recording history.

## The one real exception

A genuine `@deprecated` JSDoc tag, `// deprecated:` comment, or `##
Deprecated`/CHANGELOG `Deprecated` entry on something **actually** deprecated
in the real code today. Describe what replaced it and why, in present tense,
same as any other doc/comment — this is a normal, expected convention, not
the narrative the rule above forbids. The test: is the symbol/behavior
itself marked deprecated in code right now? If not, it doesn't qualify —
"this changed recently" is not the same claim as "this is deprecated."

## Checklist

- [ ] English throughout — no Spanish (or other non-English) slipped in from
      the authoring session.
- [ ] Present tense — describes current behavior, not authoring history.
- [ ] No reference to a session, a plan, or a phase/stage of work.
- [ ] Any `@deprecated`/deprecation notice matches something genuinely
      deprecated in the real code today, not a stand-in for "this changed."
- [ ] No rationale duplicated where a link would do — check whether this
      exact justification already lives on another symbol before writing it
      again, in this file or a sibling one.
- [ ] No real consumer/client app or org-specific package/repo name (e.g. a
      customer's own `@org/pkg`) named in shipped docs — describe the
      failure or behavior generically instead.
- [ ] An edited comment states the current fact once — no leftover caveat or
      sub-section from a prior version still sitting next to the new one.

---
name: zanix-issue-reporting
description: When and how to file a real GitHub issue via `zanix report-issue` — repo-detection (claude-skills vs. the owning package), the local anti-JSR-drift invocation form, the title/body/label shape, and the three-bucket rule for what actually gets reported vs. what stays a chat-only note. Use whenever an agent finds something real it isn't fixing in the same change — a stale skill/agent claim, a bug/footgun found incidentally, or a design question that needs human review.
---

The durable half of this ecosystem's maintenance-feedback loop (see
`skill-and-agent-authoring` for the authoring half). Filing an issue needs
no repo write access — only a token — so this applies the same way to a
maintainer agent working inside a Zanix library repo and a consumer-side
agent working inside someone's own app; there's no audience restriction on
who can report.

## Golden rule (token savings)

- **Report, don't accumulate.** A finding that isn't fixed in the same
  change either gets filed now or it's lost — don't let it pile up as
  unwritten "I should mention this" state, and don't let a skill file's own
  prose become the permanent home for it either (see
  `skill-and-agent-authoring`'s note on transient state).
- **Not everything gets filed — check the bucket first**, below. Filing an
  issue for something that's correctly out of scope (not a defect, just a
  different agent's job) is noise, not signal.
- **Confirm before filing when a human is right there** (the interactive
  reporting flow) — filing is a real, outward-facing, visible action.
  **Don't ask when nothing is fixed in the same run** (an agent's own
  automatic report at the end of its work) — that's the entire point of the
  automatic mode. **Exception: a periodic sweep agent batches instead of
  filing per-finding automatically** — see below.

## The three buckets — what actually gets reported

| Bucket | Shape | Report it? |
| --- | --- | --- |
| **A — real bug/gap/staleness** | A skill/agent's own claim no longer matches reality; a security-relevant footgun or real defect noticed incidentally, not fixed in the current change; a drift mismatch not fixed inline. | **Yes** — labels `bug`/`skill-staleness`/`agent-staleness`/`gap` as fits. |
| **B — cross-repo/cross-agent scope boundary** | The agent correctly declining out-of-scope work and naming who owns it (e.g. "that trigger action belongs in `datamaster`'s own repo," "that needs router/data-fetching, it belongs in the Application layer") — not a defect. | **No** — stays a chat-only note. Filing an issue for "this isn't my job" is noise. |
| **C — design/product question** | Something that needs a human decision before it's built at all (a genuinely new auth mechanism category, a genuinely new delivery channel) — "raise it, don't build it unreviewed." | **Yes** — labels `discussion`/`proposal`. |

When unsure which bucket a finding is, default to NOT filing and say so in
the chat report instead — a missed report is recoverable (ask again later,
or a human notices); a spurious issue on a real, public repo isn't free to
undo.

## Periodic sweep agents: batch-confirm once per run, not per finding

**Applies to**: `ecosystem-maintenance`, `conventions-sweep`,
`dependency-direction-sweep`, `dead-code-sweep`, `benchmark-sweep`,
`dependency-drift`, `documentation-agent` — agents whose own job is a
periodic/full-ecosystem sweep, not a single builder/reviewer task. Every
other agent's automatic mode is unchanged by this section: a builder or
reviewer normally surfaces 0-2 incidental findings per task, where
file-immediately-without-asking is the right call this skill's Golden Rule
already makes.

**Why this needs a different rule, confirmed real, not hypothetical**: a
real validation run of `ecosystem-maintenance` across all 12 repos produced
~20+ real, legitimate Bucket-A findings in one invocation. Filing all of
them immediately, with zero visibility and zero checkpoint, is the same
"spurious issue isn't free to undo" risk the Golden Rule already names —
just multiplied by volume instead of by uncertainty. `zanix report-issue`
now deduplicates against OPEN issues before filing (exact-title match —
see "Deduplication" below), which closes the specific "files the same
still-unfixed finding again on every repeat run" failure mode, but the
batch-then-confirm discipline below still stands on its own merits: a
20-finding batch with zero visibility is still the wrong shape even when
none of them happen to be exact repeats.

**The rule**: a periodic sweep agent gathers every Bucket-A/C finding for
the WHOLE run first, same as always — but does NOT call `zanix report-issue`
inline as it goes. At the end, the consolidated batch (repo, title,
one-line summary, per finding) is presented as part of the final report,
and filing waits for one explicit go-ahead in response to that report —
not once per finding, once for the whole batch. If no confirmation comes
(the run is unattended, or the conversation just ends there), **nothing
gets filed** — silence is never read as consent to publish. Once
confirmed, file every approved finding (or the explicitly-approved subset,
if some are called out to skip) using the normal invocation/title/body
rules below, unchanged.

This is deliberately safe-by-default for unattended/scheduled runs: an
agent dispatched with nobody present to confirm simply reports its batch
and stops there, rather than either blocking forever or filing unreviewed.

## Repo detection

- The finding is about a skill or agent file's own accuracy/completeness →
  `--repo claude-skills`.
- The finding is a real bug/gap in a package's own code, noticed
  incidentally while doing unrelated work → `--repo <owning-package>` (the
  real `zanix-io/<name>` repo — confirmed real remotes exist for
  `datamaster`/`server`/`cli`/`notifications`/`auth`/`asyncmq`/`admin`/
  `core`/`app`/`space-ui`/`utils`; `space` has no remote configured as of
  this writing — fall back to `claude-skills` and say so explicitly if that
  repo turns out to have no real target yet).
- Genuinely unsure which package owns it → default to `claude-skills`,
  never guess a specific wrong package repo.

## Invocation — prefer local, but don't assume it exists

Same reasoning `zanix-feature-builder` already established for every other
CLI invocation: a global `znx`/`zanix` install can point at an
already-published JSR version drifted against the real, current `cli`
checkout — real risk for a MAINTAINER session working inside the Zanix
monorepo. But this skill explicitly applies to consumer sessions too (see
the intro) — a consumer working in their own app has no
`~/Documents/Development/ZanixLibraries/cli` checkout at all, and assuming
one exists unconditionally is a real bug, not a hypothetical edge case.
Resolve which form to use, in this order, checking for real rather than
guessing:

1. **Local `cli` checkout exists** (`[ -d ~/Documents/Development/ZanixLibraries/cli ]`
   succeeds) — maintainer case, freshest source, no JSR-drift risk:
   ```
   (cd ~/Documents/Development/ZanixLibraries/cli && deno task znx report-issue \
     --repo <name> \
     --title "<short, specific title>" \
     --body-file <path-to-a-temp-file-with-the-real-body> \
     --label <label> [--label <label> ...])
   ```
2. **No local checkout, but a global `zanix`/`znx` binary is installed**
   (`command -v zanix` or `command -v znx` succeeds) — the real consumer
   case, a real installed CLI, same flags:
   ```
   zanix report-issue --repo <name> --title "<short, specific title>" \
     --body-file <path-to-a-temp-file-with-the-real-body> \
     --label <label> [--label <label> ...]
   ```
3. **Neither exists** — don't invent a command that will just fail. Tell
   the caller the CLI needs to be installed first, with the real install
   command (`cli`'s own README "Installation" section has the full set of
   options, including shell/PowerShell installer scripts):
   ```
   deno install -A -g -n zanix jsr:@zanix/cli@<version>
   ```
   Then retry once it's confirmed installed — don't silently skip
   reporting just because step 1/2 didn't resolve.

Prefer `--body-file` over inline `--body` for anything beyond a one-line
finding — write the real file:line citation(s), what's wrong, what's
expected, and the precedent it contradicts (the same "handoff prompt" shape
this ecosystem already uses for cross-session findings), to a temp file
first.

**When the caller will paste the command by hand** (the "Auth" section's
standing recommendation below), write that temp file to a short, flat path
— e.g. `/tmp/zanix-issue-<slug>.md` on macOS/Linux — not this harness's own
deep per-session scratchpad path (`/private/tmp/claude-<pid>/<project>/
<session-uuid>/scratchpad/...`). **Confirmed real, 2026-08-23**: that nested
path is long enough that a real terminal soft-wrapped it mid-paste,
splitting the `--body-file` argument across two lines and producing a
"No such file or directory" the human had no way to self-diagnose. This
only applies to the human-paste flow — when an agent/session runs the
command itself via `Bash` (no paste involved), the normal scratchpad
convention is fine and preferred, since nothing has to survive a terminal's
line-wrapping. `/tmp` is confirmed working on macOS; not yet verified on
Windows — if the target machine has no `/tmp`, fall back to a short path
under the user's home directory instead.

## Title/body shape

- **Title**: `[<bucket-label>] <repo-or-skill-name>: <short, specific
  description>` — e.g. `[skill-staleness] zanix-envvar-conventions: SEARCH_ENGINE_ENV
  precedent citation is stale`, `[bug] cli: generate handler --type graphql
  mis-names its context parameter`.
- **Body**: real file:line, the actual claim/behavior vs. what's true/
  expected, and — when known — which agent or skill should own the fix.
  Don't paraphrase a finding you haven't verified; state exactly what was
  checked and how.

## Auth

`zanix report-issue` reads `GITHUB_TOKEN` from the environment itself — this
skill doesn't manage auth, only when/what/where to report. If the command
fails on a missing/invalid token, surface that error to whoever's reading
the report; don't silently swallow a failed report as if it succeeded.

**Confirmed real, 2026-08-21 — the full pipeline (repo detection, local
invocation, multi-label filing, dedup) is proven working end-to-end**
against real GitHub API calls (3 real issues filed across all 3 buckets,
plus a confirmed real duplicate-skip). Getting there also settled the
token question for good — 3 different ways of getting a token from the
user into a command run through the `!` relay were tried, and **none of
them avoid the token appearing in this conversation's transcript**: a
`!`-exported env var doesn't even persist into a later Bash call
(`${#GITHUB_TOKEN}` = 0, confirmed); a file the user writes via `!` IS
genuinely readable afterward, but the `!` mechanism still echoes the full
write command — including a literal token argument — back into the
transcript regardless of where it's headed; `read -s` (hidden input,
which would avoid a literal argument entirely) fails outright through
`!` (`no coprocess`, no real TTY attached).

**The standing recommendation, settled**: **have the user run the actual
`zanix report-issue` command themselves, entirely in their own terminal —
never relayed through `!`.** Give them the exact command (repo, title,
body-file path, labels); they report back only the resulting issue URL.

**Never include an `export GITHUB_TOKEN=...` line in that command**, not
even as a placeholder for the human to fill in — until this harness
supports real secrets management, an `export GITHUB_TOKEN=<token>` line
handed to the user trains exactly the workflow this skill exists to avoid:
one bad paste (an error message, a "here's what I ran" copy-back) away from
a live token landing in the transcript anyway, defeating the whole point of
keeping the token out of `!`. State the token as a prerequisite in prose
instead — "make sure `GITHUB_TOKEN` is already set in this terminal
(`echo $GITHUB_TOKEN` to check), or export it yourself before running this"
— and let the command itself assume it's already there. Recommend a classic
PAT (`public_repo` scope for a public repo, `repo` for a private one) over a
fine-grained one when they create it — a fine-grained token is scoped to
exactly one Resource Owner at creation, so a personal-account token's repo
picker won't even list an org-owned repo (even for a direct collaborator),
and an organization can additionally restrict fine-grained PAT access
outright (confirmed true for this ecosystem's own org) — classic tokens
have neither limitation.

If a live token genuinely must pass through this session anyway (e.g.
testing/debugging the pipeline itself, as today), treat it as
unavoidably transcript-exposed and mitigate with a short-lived,
narrowly-scoped token revoked immediately after use — a real, useful
mitigation, just not a substitute for the standing recommendation above.

## When a DISPATCHED agent can't file it itself

**Confirmed real, not hypothetical**: a dispatched sub-agent (any agent run
via the `Agent` tool — a periodic sweep, a builder, a reviewer) has no more
access to `GITHUB_TOKEN` than the top-level session does on a fresh
checkout — confirmed empirically (`$GITHUB_TOKEN` unset by default; a
`!`-relayed export from the user's own terminal doesn't persist into any
Bash call, sub-agent or not, per "Auth" above). In practice this means a
dispatched agent's own attempt to run `zanix report-issue` almost always
fails on a missing token — **the agent is strictly a detector/intermediary
for these findings, not the one that actually publishes them.**

**The guarantee**: when an agent hits a real Bucket-A/C finding and can't
file it (no token, or any other failure), its final report MUST include
the exact, ready-to-run `zanix report-issue` command — real `--repo`, real
`--title`, a real `--body-file` path pointing at content the agent already
wrote to disk, real `--label` flags — not a prose paraphrase of the
finding, and not a placeholder telling the reader to "construct the
command yourself." Whoever reads that report (the dispatching session, or
the human on the other end of it) copies the command as printed, exports
their own token, and runs it — zero re-derivation needed. This is the only
way a finding surfaced by a dispatched agent actually reaches a filed
issue; if the report only describes the finding in prose, the command has
to be reconstructed by hand later, which is exactly the kind of "I should
mention this" state the Golden Rule already warns against losing.

**The same "no gate" rule applies one level up, at the dispatching
session**: when relaying a sub-agent's report to the human, show the
ready-to-run command directly in that same message — never ask "do you
want me to show you the command?" first. The command is already built;
withholding it behind an extra confirmation question is the exact
"unwritten I should mention this" friction the Golden Rule already warns
against, just moved one hop later. Asking what to do NEXT with the
command (file it yourself, run it in your own terminal, dispatch
`zanix-issue-reporter` interactively) is fine and often useful — asking
permission to even see it is not.

## Checklist before filing

- [ ] Classified against the three buckets — not filed on reflex because
      *something* was noticed?
- [ ] Correct `--repo` — a skill/agent finding never goes to a package repo,
      a package bug never goes to `claude-skills`?
- [ ] Invoked locally (`deno task znx` from the `cli` checkout), never a
      bare `zanix`/`znx` that might resolve to a stale global install?
- [ ] Title specific enough to be found later, body has the real
      file:line — not a vague restatement of "something seems off"?
- [ ] (Periodic sweep agents only) Batch confirmed once for the whole run
      before filing anything — not filed per-finding as it's found?
- [ ] Couldn't file directly (no token)? The final report has the exact,
      ready-to-run command — not a prose description the reader has to
      turn into a command themselves?

## Deduplication

`zanix report-issue` checks for an existing OPEN issue with the exact same
title in the target repo before filing — if one's found, nothing new is
created; the command reports the existing issue's real URL instead (loud,
not a silent no-op). This is `@zanix/cli`'s own implementation detail
(`commands/report-issue/lib/github-issue.ts`), not something this skill
re-documents in full — the practical effect for anyone using this skill:
re-reporting the same finding with the same title on a later run is safe
and won't spam a duplicate issue, so don't hand-roll your own dedup check
before invoking the command.

## Out of scope — do not do these

- Deciding whether a Bucket-A finding should be fixed immediately instead
  of filed — if it's fixable in the same change, fix it; only file what's
  genuinely not being fixed now.
- Filing on behalf of a Bucket-B finding to be "thorough" — that's the one
  case explicitly not covered, by design.

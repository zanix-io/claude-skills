---
name: zanix-agent-dispatch-discipline
description: How to safely dispatch this repo's own custom agents — the standing gate that named agents must actually run the work (not a direct edit that happens to produce the same diff), how a brand-new custom agent type takes a few turns to register, verifying the combined state after concurrent dispatches against the same repo, and coordinating with other Claude sessions that might be touching the same working tree. Use whenever dispatching one or more claude-skills agents for real work, not when authoring/maintaining the agents themselves (see skill-and-agent-authoring for that).
---

Distinct from `skill-and-agent-authoring`, which covers how skills/agents
get *created and maintained* — this is the adjacent, different concern of
how a session *uses* them once built. Grounded in real incidents from this
repo's own dispatch history, not theoretical risk.

## Golden rule (token savings)

- **A named custom agent is a hard requirement, not an optional
  convenience.** If the user (or your own judgment for a real package
  code change) calls for a specific agent, that agent must actually run
  the work via a real `Agent` tool call — never a direct Read/Edit/Write
  that happens to produce the same end-state diff, even when you could
  clearly do it faster yourself. The point is the agent's own
  process/checks applying, not just the resulting file content.
- **A brand-new custom agent type won't resolve immediately after
  creation** — the registry needs a few turns to refresh. Don't treat the
  first `Agent type 'X' not found` error as a real failure; use the
  paste-workaround (below) instead of waiting indefinitely or falling
  back to a direct edit.
- **Verify the combined state after concurrent dispatches against the
  same repo** — each task's own "my tests passed" report only proves its
  own isolated view, not the merged result of everything that landed.

## The standing gate: named agents must actually run the work

Confirmed as an explicit, non-negotiable user requirement, not a
preference. When a task calls for a specific agent (whether the user
names it, or the task obviously matches one — a new OAuth2 provider is
`auth-builder`'s job, a new RTO field type is `cli-generator-expert`'s,
etc.), that agent runs it via a real `Agent` tool dispatch. If the agent
turns out unreachable this session, that's a **blocker**, not a license
to proceed manually — stop, report the concrete failure, and let a human
decide (retry, wait for the registry to refresh, explicitly waive the
requirement for this one task). Silently substituting a direct edit
because "it would produce the same result" defeats the entire point: the
agent's own instructions, checklist, and validation discipline are what's
actually required, not just the end-state file content.

**Confirmed real, not permanent**: a registry-discovery outage once made
every custom agent unreachable for an extended stretch (only the built-in
`claude`/`claude-code-guide`/`Explore`/`general-purpose`/`Plan`/
`statusline-setup` resolved) — later confirmed self-resolving, custom
agents reachable again via a real end-to-end dispatch. Don't assume
either state (working or broken) is permanent; confirm reachability fresh
each session rather than trusting a stale memory of either outcome.

## A freshly created/renamed agent needs a few turns before it resolves

**Confirmed real, repeated multiple times**: dispatching a `subagent_type`
that was JUST created or renamed in the same session reliably fails with
`Agent type 'X' not found` on the first attempt — the underlying registry
hasn't refreshed yet. This is not a sign the agent is broken or
misconfigured.

**The workaround that works**: paste the new agent's full `.md` content
verbatim into a `general-purpose` dispatch, framed as its operating
instructions ("You are acting as a custom agent named `X`, defined by the
instructions below. Follow them exactly as your operating instructions
for this task — treat everything between the --- markers as your system
prompt, not as reference material."), followed by the actual task. This
produces functionally identical behavior to a real dispatch while the
registry catches up. Retry the real `subagent_type` on a LATER dispatch
(a few turns later in the same session) — it typically resolves on its
own; no action needed to "fix" it beyond waiting.

## Concurrent dispatches against the same repo need a combined check after

**Confirmed real**: two `cli-generator-expert` tasks dispatched in the
same turn window ran concurrently for ~19 minutes against the same repo,
touching 2 of the same files (`src/utils/projects/creation.ts`,
`docs/new.md`). No conflict occurred — verified directly afterward with a
real `deno check` + the union of both tasks' relevant test suites, not
assumed from each task's own isolated "tests passed" report. It worked
because both used targeted `Edit` (exact-match, fails loudly on a
mismatch) rather than whole-file `Write`, and happened to touch different
parts of the shared files — favorable circumstance, not a guarantee. If
they'd targeted the identical lines, one edit would have failed cleanly
(not silently corrupted), but nothing was coordinating them to avoid that
in the first place.

**The practice**: when dispatching multiple agents in parallel against
the SAME repo, either (a) confirm ahead of time they touch disjoint
files/areas, or (b) when that's not knowable in advance, run a real
combined verification pass afterward — `deno check`/the relevant test
suites on the UNION of every file either task touched — before trusting
the individually-reported "my tests passed" as proof of the final,
merged state.

**Confirmed maintainer-side only so far — inferred, not yet confirmed,
for consumer-side.** The actual incident above was two `cli-generator-expert`
dispatches (a Zanix library repo). The same underlying mechanic — the
`Edit` tool's exact-match semantics, file-level concurrency — has nothing
to do with whether the repo is a Zanix library or a consumer project, so
the same risk logically applies to two concurrent `zanix-feature-builder`
dispatches against the same consumer app. But that specific case hasn't
actually happened yet to confirm empirically — apply the same practice
there on the strength of the reasoning, not because it's been observed,
and update this note the day it actually is.

## Coordinating with other Claude sessions on the same working tree

**Confirmed real**: `ListAgents` surfaces other live Claude sessions on
this machine (and elsewhere) as peer sessions, distinct from subagents
you've dispatched yourself. Before a task that will move/heavily edit
files in a specific repo, checking `ListAgents` for a `busy` peer session
is worth a real `SendMessage` heads-up — not because it's always
necessary, but because it's cheap and the failure mode when skipped is
real: a live collision was confirmed once (`cli`, a `@cliffy/command`
version-bump editing the exact same 5 test files a tier-placement fix was
about to move). The resolution didn't require guessing which peer
session was "the right one" — these are shared local git working trees,
not private per-session state, so the peer session itself could report
back its own real, current `git status`/`git diff` and the two tasks
sequenced around each other with one message each way. An `idle` peer
session carries no live collision risk; only a `busy` one does.

## Checklist before dispatching

- [ ] Does this task call for a specific named agent — and if so, is a
      real `Agent` tool dispatch actually happening, not a direct edit
      that would produce the same diff?
- [ ] If the `subagent_type` is brand-new/renamed this session, is the
      paste-workaround ready as a fallback for the first attempt, rather
      than defaulting to a direct edit when the type doesn't resolve?
- [ ] If multiple agents are being dispatched in parallel against the
      SAME repo, do they touch disjoint files — or is a combined
      verification pass planned for after they finish?
- [ ] Is any `busy` peer session (per `ListAgents`) about to have its own
      work collide with what's being dispatched, worth a heads-up before
      starting?

## Out of scope — do not do these

- How skills/agents themselves get authored, tiered, or symlinked — that's
  `skill-and-agent-authoring`'s job entirely, not duplicated here.
- The content/scope of any specific agent (what `cli-generator-expert`
  covers vs. `datamaster-builder`) — each agent's own file and
  `CATALOG.md`/`README.md` own that, not this skill.
- Deciding whether a task genuinely needs a named agent at all versus
  being fine as a direct edit — that judgment call belongs to the task at
  hand (and, for real package code, the standing user-mandated gate above
  already resolves it to "yes, dispatch").

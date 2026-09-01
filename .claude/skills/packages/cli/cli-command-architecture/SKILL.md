---
name: cli-command-architecture
description: How @zanix/cli's command tree is wired with Commander/cliffy — command groups, mountGroup, and the registration-function convention every leaf command follows. Use before adding a new top-level command group, or when a command's error handling/help output looks wrong.
---

This skill is about the plumbing that wires `zanix <group> <leaf>` together —
not about what any generator or scaffold actually produces. For that, see
`cli-artifact-generators` (`zanix generate`) and `cli-scaffold-assembly`
(`zanix new`). File:line references point at
`~/Documents/Development/ZanixLibraries/cli` — read the real code there before
assuming this summary is still accurate; it will drift as the CLI evolves.

## Golden rule (token savings)

- This skill's rules are checked by inspection, not by running anything — a
  `mountGroup` mistake shows up as a raw error message the first time the
  group's command errors, not something to reproduce experimentally before
  flagging it. Cite the rule and the file:line it lives at; don't spin up a
  manual repro to prove a documented framework constraint.

## Command groups: always `Commander.mountGroup`, never raw `.command()`

Every one of the 8 top-level commands (`new`, `generate`, `build`, `prepare`,
`space`, `report-issue`, `check-cycles`, `credentials`) is built the same way:
a bare `new Commander()` pseudo-parent (`cwd`) is mounted onto `cli` via
**`Commander.mountGroup(name, cwd)`** — never the raw `Command.command(name,
cwd)`. Most (`new`, `generate`, `prepare`, `space`, `credentials`) then
register real sub-leaves on `cwd` (`space dev`/`space build`, `credentials
mesh`, ...); `build`/`report-issue`/`check-cycles` have no sub-leaves at all —
`cwd` itself carries the one `.action()`, same mount call either way.

This isn't dead code, even though it's currently redundant. As of
`@cliffy/command@1.2.1` (the version this repo pins), `getErrorHandler()`
already walks the whole parent chain natively, so a leaf command mounted any
number of levels below `cli` finds `cli`'s own `.error()` handler on its own
without `mountGroup`'s help. The re-application is kept anyway as a
deliberate, low-cost belt-and-suspenders safeguard against a future cliffy
release reintroducing the single-level lookup it had in `1.0.0-rc.8`, using
only cliffy's own public `.error()`/`.throwErrors()` API — no private-field
access, no patching the dependency.

**Skip it for a future command group anyway.** At the currently pinned cliffy
version the native chain-walk covers the gap, but `mountGroup`'s
re-application is the only thing standing between that group's leaves and a
regression back to the single-level lookup cliffy used before `1.2.1` — skip
it and that regression silently degrades the group's error UX (help text + a
clean message) to a raw, unformatted rejection, with no local signal that
anything changed. `mod.ts`'s own top-level boundary still catches it, so it's
never *silent* in the sense of swallowed — just ugly, and inconsistent with
every other command's error output.

## Leaf commands need a real `Commander` instance, not cliffy's bare `Command`

Leaf commands built through `baseArgumentActionCommand` (`utils/commands.ts`)
are real `Commander` instances too — passed in explicitly as
`this.command(name, new Commander())` — never cliffy's own bare `Command` its
default `this.command(name)` would create.

That's what makes `this.runCommand(...)` callable at all from inside a leaf's
own action, where `this` is that leaf — used by every `zanix new <type>`'s
automatic `--prepare` step, which needs to invoke `prepare`'s own command tree
from two levels below `cli` (`cli -> new -> <type>`). `runCommand` itself walks
up to the true root rather than assuming a single parent hop, since a leaf is
two levels below `cli`, not one. Building a leaf with a bare `Command` breaks
this the same way skipping `mountGroup` breaks error handling — same root
cause (cliffy's own APIs assuming a shallow parent chain), two different
symptoms.

## Registration pattern: each command owns its full chain

Each command's registration function (`register<Name>Command(cwd: Commander):
void` for a generator, the equivalent for any other leaf) owns its full
`.command().description().arguments().option()?.action()` chain directly — no
shared generic helper tries to thread `.option()` through a loop. cliffy's
`.option()` builds a per-call, incrementally-narrowed generic type, and a
shared helper applying it generically breaks that inference (a real,
discovered-the-hard-way constraint, not a style preference). A command with no
options can still use a thin registration helper; a command needing options
(`rto`'s `--field`, `job`'s `--cron`, `credentials password-hash`'s
`--level`/`--var-name`) registers its full chain directly. Not
generator-specific — `credentials password-hash` is a plain top-level leaf,
not a `generate` artifact, and follows the identical pattern, confirming
this rule is about cliffy's own `.option()` inference, not anything
particular to a generator's own module shape. See `cli-artifact-generators`
for the full module-layout convention this pattern additionally lives
inside for generators specifically.

## Checklist before adding a new command group or leaf

- [ ] Is this a new top-level command (like `new`, `generate`, `build`,
      `prepare`, `space`, `report-issue`, `check-cycles`, `credentials`)? It
      must be mounted via `Commander.mountGroup`, never `Command.command`, or
      its errors (or its leaves' errors, if it has any) silently lose the
      polished CLI error UX.
- [ ] Does this leaf (or anything it calls) need `this.runCommand(...)` to reach
      another command tree? It must be built as a real `Commander` instance via
      `baseArgumentActionCommand`, not cliffy's bare `Command`.
- [ ] Does this command need options? If yes, register its full chain directly
      in its own function — don't thread `.option()` through a shared generic
      helper.

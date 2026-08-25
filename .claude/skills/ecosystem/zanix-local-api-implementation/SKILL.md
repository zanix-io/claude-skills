---
name: zanix-local-api-implementation
description: The concrete mechanics for building a new local-API subpath — export-subpath structure, the Controller/guards composition pattern, and the real dependency-boundary test technique. Use when actually writing a new <domain>-api/ subpath. Not for deciding whether a resource needs a local API vs an aggregator in the first place — see zanix-local-api-vs-aggregator for that decision.
---

This is the how-to companion to `zanix-local-api-vs-aggregator` — that skill is
the rule (does this resource need a local API at all, and does this controller
belong there or in `@zanix/admin`); this skill is the mechanics of actually
building one once the answer is yes. Don't start here — confirm the need with
the rule skill first. Grounded in `@zanix/space`'s `assets-api`, the most
complete reference implementation in the ecosystem. File:line references point
at `~/Documents/Development/ZanixLibraries/space` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- Copy the structure from `assets-api` directly (subpath layout, controller
  shape, the dependency-boundary test file) rather than re-deriving it from this
  skill's prose each time — the test file itself is short enough to copy and
  adapt wholesale for a new domain.
- Run the dependency-boundary test's `deno info --json` check once per new entry
  point added, not repeatedly while iterating on unrelated logic in the same
  controller.

## Subpath structure

A local API is its own `deno.json(c)` exports subpath, never re-exported from
the package's main entrypoint (`.`) or folded into the domain module it fronts.
Real example, `@zanix/space/deno.jsonc`:

```jsonc
// The Asset Application/HTTP API — a NEW, strictly one-way layer ABOVE `./assets`: it imports
// from `modules/asset-transform/`, never the reverse (enforced by
// `src/@tests/unit/asset-transform/dependency-boundary.test.ts`). Real `@Controller`/
// `ZanixController` REST routes ... over `AssetStorage`/`AssetRepository`/`JobDispatcher` ports —
// Mongo/S3 are NOT implemented here, only the ports a future `@zanix/datamaster` adapter
// implements. `modules/assets/`, `modules/media/`, and `modules/asset-transform/` themselves
// never import `@zanix/server` — only this subpath does.
"./assets-api": "./src/modules/assets-api/mod.ts",
```

The comment isn't decoration — it states the layering rule inline, at the one
place a future maintainer is guaranteed to look when adding a new subpath. Write
an equivalent comment for a new local-API subpath, not just the bare export
line.

## Controller composition: the domain service is injected, never constructed internally

```ts
export interface AssetsControllerOptions {
  /** The composed `AssetService` every route delegates to. */
  service: AssetService;
  prefix?: string;
  guards?: { write?: MiddlewareGuard[]; read?: MiddlewareGuard[] };
}
```

The controller takes its domain service as a constructor/factory option — it
never imports the concrete implementation (ffmpeg-backed transformers, storage
connectors) itself. This is what the dependency-boundary test below actually
proves, and it's what keeps the controller genuinely swappable and free of the
domain layer's own heavy dependencies (a real concern here: the controller must
never _really load_ ffmpeg/sharp at runtime, only reference the service's type).

## Guards: default to deny-all, not "no guard"

```ts
function combineGuards(guards: MiddlewareGuard[] | undefined): MiddlewareGuard {
  const list = guards && guards.length > 0 ? guards : [denyAllGuard];
  return async (context, ...args) => {
    for (const guard of list) {
      const result = await guard(context, ...args);
      // ...short-circuit on first denial
    }
  };
}
```

**`assets-api` defaults every guard group to `[denyAllGuard]` when omitted —
"never public by accident."** This is the pattern to follow for a _new_ local
API.

**Caution — the ecosystem is not consistent on this today.** Datamaster's
`createTriggersAdminController` and the equivalent `templates-api` controller
both document themselves as _"omit `guards` for no auth at all — this package
assumes none by default."_ That's the opposite default from `assets-api`'s
deny-all. Don't copy the "no guard by default" pattern into a new local API just
because an existing reference implementation does it that way — `assets-api`'s
deny-all is the safer default and the one to build new work on. If you're
touching an existing controller that already defaults to no-auth, that's a
candidate finding for a real security review, not something to silently
propagate into a new one.

**This doesn't contradict `zanix-server-internals`'s "no guard, not a fake one"
rule for `@zanix/server` kernel primitives** — the two apply at different
layers, and reconcile once made explicit. That rule is about never _pretending_
to protect something (a guard that looks like it checks permissions but
doesn't); `denyAllGuard` isn't fake in that sense — it's a real, honest guard
that does exactly what it says, deny everything, reliably. The actual difference
is which failure mode each layer accepts when nothing is configured: a kernel
primitive that owns no specific resource of its own (Discovery, or a future one)
fails open — honestly unauthenticated, since there's no real data behind it to
protect by default. A local-API controller fronting a package's actual business
data fails closed instead, since the cost of an accidentally-public real
endpoint is categorically higher than the cost of a real endpoint nobody wired a
guard onto yet. Don't import `@zanix/server`'s "empty guards array is honest"
framing into a local-API context as license to skip `denyAllGuard`'s fail-closed
default — the two rules are about different things (never fake protection, vs.
which side to fail on when nothing is configured), not competing answers to the
same question.

## The dependency-boundary test — the real reusable template

Real, copy-adaptable source (from
`space/src/@tests/unit/assets-api/dependency-boundary.test.ts` — read it
directly for the full file, this is the technique, not the whole listing):

```ts
async function rawInfo(entry: string) {
  const command = new Deno.Command(Deno.execPath(), {
    args: ["info", "--json", entry],
    stdout: "piped",
    stderr: "piped",
  });
  const { stdout, stderr, success } = await command.output();
  if (!success) {
    throw new Error(
      `'deno info --json ${entry}' failed: ${new TextDecoder().decode(stderr)}`,
    );
  }
  return JSON.parse(new TextDecoder().decode(stdout));
}
```

Two distinct checks, built on that raw output — **use the right one for the
claim being proven, they are not interchangeable**:

- **`moduleGraph(entry)`** — a flat union of every `code`/`type` edge across the
  _whole_ graph. Correct for proving "this entry has no type-only edge into
  something that itself pulls in a forbidden package as code" — but **not sound
  in general**: a module reachable from `entry` only via a _type_ edge still
  contributes its own _code_ edges to this flat set, which can wrongly implicate
  `entry` for a dependency it never actually loads at runtime.
- **`codeReachableFrom(entry)`** — the sound version: a real BFS from `entry`'s
  resolved root, following **only `code` edges** — exactly what a real
  `deno run`/bundler would actually load. A module reachable from `entry` _only_
  through a `type`-only edge
  (`import type { AssetService }
  from '../asset-service.ts'`) is correctly
  **excluded**, along with everything that module's own code imports (ffmpeg
  included) — because none of it is ever really loaded when only the controller
  itself runs.

Three real claims this pattern proves for `assets-api`, as templates for a new
local API's own test file:

1. **The domain implementation is referenced only as a type, never as code** —
   proves the controller's `service` is genuinely injected, not internally
   constructed.
2. **Heavy runtime dependencies (ffmpeg/sharp transcoders) are never really
   loaded** — via `codeReachableFrom`, not the coarser `moduleGraph`, since this
   is exactly the case where the distinction matters (the domain module IS
   type-reachable, and its own code edges would falsely flag it under the flat
   check).
3. **A forbidden package (`@zanix/datamaster`, or any concrete storage/
   persistence implementation) never leaks into the published subpath, at
   compile time or runtime** — checked at every real export entry point, not
   just one.

Don't invent a different verification mechanism (a `grep` over `deno.json`'s
`imports` map, a manual review) — this is the mechanism the ecosystem has
already converged on, and it's real infrastructure
(`Deno.Command('deno',
['info', '--json', ...])`), not a proposal.

## `versionProtocol`: composed by the caller, not owned by the local API

```ts
createTriggersAdminController({
  guards: [jwtValidationGuard],
  versionProtocol: { version: 1 }, // passed straight through to @Controller
});
```

A local API accepts `guards`/`versionProtocol` as factory options and passes
them through to the underlying `@Controller` — it never invents its own auth or
protocol-versioning mechanism internally (see `zanix-server-internals`'s "auth
is never assumed" rule, which this is a consumer-side instance of).

## Checklist before building a new local API

- [ ] Confirmed via `zanix-local-api-vs-aggregator` that this resource actually
      needs a local API — not an aggregator, and not something that belongs in
      `@zanix/admin` just because that's historically convenient?
- [ ] Is the subpath exposed as its own `deno.json(c)` exports entry, with an
      inline comment stating the layering rule — not folded into `.` or the
      domain module?
- [ ] Does the controller take its domain service as an injected option, never
      importing the concrete implementation itself?
- [ ] Do guards default to deny-all when omitted — not "no auth by default,"
      regardless of what an existing reference implementation does?
- [ ] Is there a `dependency-boundary.test.ts` proving the domain layer never
      imports the `-api` subpath back, and the `-api` subpath never leaks a
      forbidden package (at compile time or runtime) into its own public
      exports?
- [ ] For a claim about what's really _loaded_ at runtime (not just referenced
      structurally), is `codeReachableFrom`'s BFS used — not the coarser flat
      `moduleGraph` check, which can produce a false positive when a heavy
      dependency is only type-reachable?
- [ ] Does `guards`/`versionProtocol` pass straight through to `@Controller`,
      rather than the local API inventing its own auth/versioning shape?

## Out of scope — do not do these

- Deciding whether a resource needs a local API at all, or whether a given
  controller belongs there vs. in `@zanix/admin` — that's
  `zanix-local-api-vs-aggregator`'s job entirely; confirm with that skill
  first rather than re-deriving the decision here.
- General cross-package dependency-direction rules (tier hierarchy, when a
  direct import is valid vs. when inversion is required) —
  `zanix-dependency-direction`'s job; this skill only covers the one
  specific boundary a local-API subpath enforces against its own domain
  module.
- Auth/guard semantics for `@zanix/server` kernel primitives that own no
  specific resource of their own (Discovery, etc.) —
  `zanix-server-internals`'s "no guard, not a fake one" rule; this skill's
  deny-all default applies specifically to local-API controllers fronting
  real business data, a different layer with a different failure mode.
- Fixing an existing local API that already defaults to no-auth (e.g.
  datamaster's `templates-api`/`createTriggersAdminController`) — that's a
  candidate finding for a real security review, not something to silently
  patch as a side effect of building an unrelated new subpath.

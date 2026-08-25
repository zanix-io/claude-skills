---
name: app-sandboxing
description: The sandbox declaration (metaUrl/permissions/timeout) for running an operation inside a dedicated, permission-restricted Deno Worker — what the boundary actually protects (access, never CPU/memory), the permissions-object-replaces-not-merges gotcha, the unstable worker-options requirement's deploy-time footgun, and the structural inability to receive a live RuntimeContext. Use when an operation needs isolation from the main process, or reviewing an existing sandboxed operation's permission grant.
---

Covers `operations.<name>.sandbox` — running one operation inside its own
Deno Worker instead of inline in the main process. For everything else
about operations (declaring, authorizing, exposing to MCP), see
`app-remote-calls-and-control-plane`/`app-mcp-composability`. File:line
references point at `~/Documents/Development/ZanixLibraries/app` — read the
real code there before assuming this summary is still accurate.

## Golden rule (token savings)

- Set `permissions` explicitly on every sandbox declaration — omitting it
  gives thread isolation only, not the access restriction the feature name
  implies.
- `timeout` (default 10000ms) is the ONLY protection against a runaway
  task — there is no CPU/memory quota to reach for instead.

## Declaring a sandboxed operation

```ts
// tasks/fetch-rate.ts — a real, standalone, independently-importable module
export function fetchRate(payload: { pair: string }) {
  return fetch(`https://api.example.com/rate/${payload.pair}`).then((r) => r.json())
}
```

```ts
defineZanixApp({
  name: 'billing',
  operations: {
    fetchRate: {
      sandbox: {
        metaUrl: new URL('./tasks/fetch-rate.ts', import.meta.url).href,
        permissions: { net: ['api.example.com'], read: true, write: false, env: false, run: false },
        timeout: 5000,
      },
    },
  },
})
```

`metaUrl` is required — the task module (a real, standalone module,
independently importable — this is the task module's own `import.meta.url`,
never the caller's). `taskName` defaults to the operation's own manifest
key. `timeout` defaults to `WorkerManager`'s own default, currently
`10000`ms.

Under the hood, one dedicated `WorkerManager({ pool: 1, permissions })` is
built per sandboxed operation, fixed for its whole lifetime, tracked
per-app so `closeSandboxedWorkers(appName)` can terminate them all (called
automatically by `uninstallApp`, `app-hot-install-and-multitenancy`). The
task function is never imported into the host process — it's loaded via a
dynamic `import(metaUrl)` inside the worker itself.

## What the boundary actually protects, and two real footguns

The `sandbox` declaration's `permissions` field is passed straight through
to a dedicated `WorkerManager` (see "Declaring a sandboxed operation"
above) — every real footgun about what `permissions` protects (access
only, never CPU/memory), how it's evaluated (replaces the whole set,
doesn't merge with an implicit default-allow), and the unstable
`worker-options` requirement's deploy-time failure mode is `WorkerManager`'s
own, documented once in `@zanix/utils`'s `utils-workers` — read that
skill for the full detail rather than re-deriving it here. The one thing
worth restating in this package's own terms: **omitting `permissions`
gives thread isolation only, not the access restriction "sandboxing" as a
feature name implies** — still real (a runaway task can't take down the
main process), but easy to mistake for full sandboxing when it isn't one
without `permissions` explicitly set.

## Structural limitation: no live `RuntimeContext`

A Worker communicates only via `postMessage` (structured-clone) — a
sandboxed operation can **never** receive a live `RuntimeContext`.
`ctx.resource()`/`ctx.remote()` are unavailable inside it, not just
discouraged; the handler silently ignores whatever `ctx` it's called with
and only forwards `payload`. Design a sandboxed task's inputs/outputs as
plain, structured-clone-safe data — never assume it can reach back into the
host process for a resource or a remote call.

## Errors

Any failure inside a sandboxed operation — permission denial, a
thrown/rejected task, or a timeout — always surfaces uniformly as
`InternalError SANDBOX_TASK_FAILED`, wrapping the underlying error as
`cause`. `ctx.remote(...).call(...)` never distinguishes a sandboxed
failure mode from a regular one at the call site — inspect `cause` if the
distinction matters.

## Checklist before adding/reviewing a sandboxed operation

- [ ] Is `permissions` set explicitly — not omitted under the assumption
      that "sandbox" alone means restricted access?
- [ ] Does `permissions` include everything the task module itself needs
      to even load (`read`/`net` for `metaUrl`), not just what the task's
      own logic needs at runtime?
- [ ] Is `"unstable": ["worker-options"]` actually enabled in this
      project's `deno.json` if `permissions` is set — checked at
      deploy/CI time, not discovered under production load?
- [ ] Does the task's own module design assume it can only ever receive
      plain, structured-clone-safe `payload` data — never a live
      `ctx.resource()`/`ctx.remote()`?

---
name: utils-workers
description: WorkerManager — task registration/invocation, the permissions option (replaces the whole set, doesn't merge with a default-allow), the unstable worker-options requirement, and the honest limitation that it protects access, never CPU/memory. Use when running a function inside a dedicated Deno Worker, with or without permission restriction.
---

Covers `@zanix/utils/workers`'s `WorkerManager` — the underlying primitive
`@zanix/app`'s operation sandboxing (`app-sandboxing`) and `Logger`'s
`useWorker` storage style (`utils-logger`) both build on. File:line
references point at `~/Documents/Development/ZanixLibraries/utils` — read
the real code there before assuming this summary is still accurate.

## Golden rule (token savings)

- Set `permissions` explicitly whenever real isolation is the goal —
  omitting it keeps the full host process permission set, an
  insecure-by-default posture worth checking deliberately, not assuming.
- `timeout` (default 10s) is the only protection against a runaway task —
  don't look for a CPU/memory quota option that doesn't exist.

## Registering and invoking a task

```ts
import { WorkerManager } from 'jsr:@zanix/utils@[version]/workers'

export function add(a: number, b: number) { return a + b }

const wm = new WorkerManager()
wm.task(add, {
  metaUrl: import.meta.url,
  onFinish: (result) => console.log(result), // { response: 5, error: null, ... }
  autoClose: true,
}).invoke(2, 3)
```

`new WorkerManager(options?, createWorker?)` — `options.pool` default `1`
worker; `createWorker` is a factory replacing the default worker
constructor, used mainly in tests (must return `{worker, status: 'free'}`).

`.task(fn, options)` registers a task and returns an object with
`.invoke(...args)`. **`fn` must be an exported function from the module at
`options.metaUrl`** (typically `import.meta.url`) — the worker dynamically
imports that module and looks the function up by name, it doesn't
serialize the function itself. `options.autoClose` (default `false`)
terminates the worker once the task finishes. `options.timeout` (default
`10000`ms) — on timeout, the pending worker is terminated, the failure is
logged, and a new worker replaces it. If the task function returns a
`Promise`, `WorkerManager` awaits it inside the worker before posting the
result back.

On any task error (a sync throw, a rejected promise, an uncaught
error/unhandled rejection inside the worker), `onFinish` still fires — with
`error` populated and `response: null` — rather than throwing in the
caller.

## `permissions`: restricts access, replaces the whole set

```ts
const sandboxed = new WorkerManager({
  pool: 1,
  permissions: {
    read: [new URL('./fetch-task.ts', import.meta.url).pathname],
    net: ['api.example.com'],
    write: false,
    env: false,
    run: false,
  },
})
```

Mirrors Deno's `Worker` `deno.permissions`/`--allow-*` shape: `true`/
`false`/`'inherit'`, or an array of allowed values for
`read`/`write`/`net`/`run`/`ffi`, plus `env` and system-info categories.

**Real footgun — the object REPLACES the whole permission set, it does not
mean "restrict only what's listed, inherit the rest."** Any category left
out of the object defaults to fully **denied**. **The task's own module
needs `read` (or `net`, for a remote `metaUrl`) listed regardless of what
the task's own logic does**, because every task is loaded via a dynamic
`import(metaUrl)` inside the worker — omitting it makes the task fail
before it ever runs, with a real permission error, not a hang.

**Omitting `permissions` entirely keeps inheriting the full host process
permission set** — the same as before the option existed. This is an
insecure-by-default posture worth flagging when reviewing a
`WorkerManager` construction with no `permissions` for a task that
processes untrusted input. A worker's permissions can never exceed its
parent's own (enforced by Deno's `Worker` API itself, not by
`WorkerManager`).

**Requires Deno's unstable `worker-options` feature**
(`"unstable": ["worker-options"]` in `deno.json(c)`, or the
`--unstable-worker-options` CLI flag) whenever `permissions` is set.
**Real deploy-time footgun**: without the flag, pool creation doesn't fail
at boot — the worker throws only at first invocation, surfacing as a
runtime error under load rather than a startup failure. Confirm the
unstable flag is actually configured, not just assumed, whenever
`permissions` is added to an existing `WorkerManager`.

## Honest limitation: access control only, never resource governance

`permissions` restricts *access* (net/read/write/env/run/ffi/sys) — it is
**not** a CPU-time or memory quota. Deno's `Worker` API has no such
governance mechanism at all. `timeout` (terminating a long-running worker)
is the *only* protection against a runaway/CPU-bound task, and there is
currently no way to cap a worker's memory from plain TS/JS without a
custom Rust-embedded Deno build. Don't describe `WorkerManager` as
protecting against resource exhaustion — it protects against unauthorized
access and unbounded wall-clock time, nothing else.

## Checklist before adding/reviewing a `WorkerManager` task

- [ ] Is `permissions` set explicitly for any task processing untrusted
      input — not left to inherit the full host permission set by
      omission?
- [ ] Does `permissions` include whatever the task's own module needs just
      to load (`read`/`net` for `metaUrl`), not just what the task's logic
      needs at runtime?
- [ ] Is `"unstable": ["worker-options"]` actually configured if
      `permissions` is set — checked directly, not assumed?
- [ ] Is `timeout` tuned for the actual expected task duration — the only
      real defense against a runaway task, given there's no CPU/memory
      quota to fall back on?

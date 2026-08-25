---
name: app-standalone-deployment
description: bootstrapRemoteApp — running one Zanix App as its own standalone remote process (not embedded via Zanix.start()), its graceful SIGINT/SIGTERM shutdown sequence, and the zanix prepare --docker -p app CLI scaffolding it plugs into. Use when deploying a single app as an independent remote process instead of embedding it in a host.
---

Covers `bootstrapRemoteApp` — the standalone-process deployment path,
distinct from the normal embedded path (`Zanix.start()`'s `apps` option,
`app-manifest-and-composition`). File:line references point at
`~/Documents/Development/ZanixLibraries/app` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- Most Zanix Apps use the normal embedded path — reach for
  `bootstrapRemoteApp` only when an app genuinely needs to run as its own
  independent remote process, not as a default choice.
- Let `zanix prepare --docker -p app` scaffold `serve.ts`/the Dockerfile
  rather than hand-writing the bootstrap sequence.

## `bootstrapRemoteApp`

```ts
import { bootstrapRemoteApp } from '@zanix/app/runtime'
import app from './mod.ts'

await bootstrapRemoteApp(app, {
  server: { rest: {} },
  remoteInstances: { endpoint: 'http://my-app:8000' }, // omit to run standalone without announcing
})
```

Internally: `activateApps([app], options.resources ?? {}, bindings,
undefined, options.remoteInstances ? {[app.name]: options.remoteInstances} :
{})`, then `bootstrapAppServer(app.name, options.server, true)`.
`remoteInstances` here is scoped to this one app — unlike `activateApps`'s
own batch-shaped `remoteInstances` parameter
(`app-remote-calls-and-control-plane`).

## Graceful shutdown

`SIGINT`/`SIGTERM` trigger `deactivateApps` (runs `onStop`, releases
resources) **before** `webServerManager.stop(servers)` — the app's own
cleanup always completes before the server itself stops accepting/serving.
Signal listeners are removed before running the shutdown sequence, so a
second signal (or a manual `stop()` call) can't double-run it. Exits
`Deno.exit(0)` on a clean shutdown, `Deno.exit(1)` if shutdown itself
fails.

## CLI scaffolding

```sh
zanix prepare --docker -p app
```

Scaffolds `serve.ts` (this exact `bootstrapRemoteApp` call — never
overwritten if the file already exists), a `serve` task in `deno.json`
(`deno run --env-file=.env <perms> serve.ts`), and a `Dockerfile`
(`CMD ["task", "serve"]`, the same base template `'server'`-type projects
use). Full detail lives in `@zanix/cli`'s own `docs/prepare.md`/
`docs/deploy.md` — this skill only covers what `@zanix/app` itself
contributes to that pipeline. Contrast: `zanix new app` scaffolds a bare
app with no `serve.ts` at all — the standalone-deployment scaffold is a
separate, later step (`zanix prepare`), not part of initial project
creation.

## Checklist before deploying an app standalone

- [ ] Does this app genuinely need its own independent remote process —
      or would the normal embedded path (`Zanix.start()`'s `apps` option)
      serve it just as well with less operational overhead?
- [ ] Is `remoteInstances.endpoint` set to the real, externally-reachable
      address other apps would use to reach this one via `ctx.remote()`?
- [ ] Is `serve.ts` generated via `zanix prepare --docker -p app` rather
      than hand-written, so it stays in sync with the framework's own
      shutdown sequence?

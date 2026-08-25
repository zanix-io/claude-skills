---
name: core-bootstrap-and-setup
description: Zanix.start()/bootstrap()/startWorker()/compose() and graceful shutdown, standalone vs. named-app SSR, named apps' shared resources/behaviors, and Zanix.setup()'s cross-cutting config (env-var-only options, error-handling config, and the real AssetService Zanix.setup({assets}) constructs). Use when bootstrapping a Zanix project, adding a named app, statically introspecting registered routes without booting a server, or configuring cross-cutting options before start().
---

Covers `@zanix/core`'s bootstrap entrypoints and `Zanix.setup()`. For the
built-in admin server (`admin: true`), see `core-admin-apis`; for how that
relates to the centralized `ZanixAdminHub`, see `core-admin-architecture`.
File:line references point at `~/Documents/Development/ZanixLibraries/core` —
read the real code there before assuming this summary is still accurate.

## Golden rule (token savings)

- Call `Zanix.setup()` before `Zanix.start()`/`Zanix.startWorker()`, always —
  its `database`/`notifications`/`dlq`/`assets` options work by setting env vars
  those packages read at their own module-import time, so calling it after has
  no effect on them.
- Trust "an already-set env var always wins" for every `Zanix.setup()` option —
  don't second-guess whether a deployment's own env var will be respected.

## Bootstrap entrypoints

```ts
import Zanix from "jsr:@zanix/core@[version]";

await Zanix.bootstrap({
  server: {
    rest: { onCreate, ...options },
    graphql: { onCreate, ...options },
    socket: { onCreate, ...options },
  },
});
```

**Graceful shutdown**: `Zanix.bootstrap()`/`Zanix.start()` trap
`SIGINT`/`SIGTERM` automatically, no opt-out — either signal runs `Zanix.stop()`
before exiting (HTTP servers drain in-flight requests via `Deno.serve()`'s own
`.shutdown()`, then connector connections close). This is what a
`docker stop`/Kubernetes pod termination (both send `SIGTERM` by default)
already gets, with no code required. Calling `Zanix.stop()` yourself still works
for any other shutdown trigger.

**Worker mode**: for a process that only runs background jobs (no HTTP servers),
use `Zanix.startWorker()` instead of `Zanix.start()`/`Zanix.bootstrap()`:

```ts
await Zanix.startWorker();
```

Loads the same cross-package core dependencies as `Zanix.start()` (including
`@zanix/datamaster`'s built-in `mail`/`request` trigger job handlers), then
delegates to `@zanix/asyncmq`'s worker bootstrap and keeps the process alive
until `SIGINT`. `Zanix.stopWorker()` closes the worker's connector connections
but does **not** itself terminate the process — call it before your own shutdown
logic exits, not as a replacement for it.

**Static introspection, no server**: `Zanix.compose(rootDir?, { admin? })` registers this
project's own decorator metadata (and, with `admin: true`, `@zanix/admin`'s built-in local admin
app + its enabled sub-apps) exactly like `Zanix.start()` does before it boots — without starting
any server or activating any real infrastructure:

```ts
await Zanix.compose(); // same rootDir default as start()/bootstrap()

const routes = ProgramModule.routes.getRoutes("rest"); // from `@zanix/server`
```

Useful for a process that only needs to read back what `start()`/`bootstrap()` WOULD register
(e.g. a static OpenAPI generator) without paying for a real boot. `admin: true` is safe to include
here (none of `@zanix/admin`'s fixed manifests declare `dependencies`/`resources`/`onStart`/
`onStop`/`jobs`, so nothing needs tearing down) — but `compose()` deliberately has no equivalent
for a project's own named `apps`: unlike `admin`'s fixed manifest set, an arbitrary `apps` entry
can declare a real dependency or `onStart` side effect, so there's no way to activate it here and
keep the "zero side effects" guarantee this function exists to provide.

## SSR: standalone vs. named apps

```ts
await Zanix.start({ server: { ssr: { port: 3000 } } }); // standalone, default 'main' Application
```

Use the standalone `server.ssr` form directly for a project with just one or a
few hand-written SSR pages, not composing a larger frontend. A composed frontend
framework built on `@zanix/app`'s manifest (`@zanix/space`, with its own
routing/hydration/PWA) registers as a **named app** instead:

```ts
await Zanix.start({
  apps: {
    storefront: { definition: spaceApp, server: { ssr: { port: 3000 } } },
  },
});
```

Both paths use the exact same `'ssr'` handler type underneath — the difference
is composition, not capability. Prefer the named-app form once the frontend has
its own manifest/lifecycle/resources to manage (that's what `apps` buys:
Application identity, independent resources, and future distributed-runtime
portability with no code change); reach for standalone `server.ssr` only when a
project never needs any of that.

**Real gotcha, no automatic port sharing**: a named app's `server.ssr` with no
explicit `port` falls back to its own default (`STATIC_PORT`), never `main`'s —
pass the identical `port` value on both sides if the two are meant to share one
listener.

## Named apps: shared resources and behavior overrides

```ts
await Zanix.start({
  resources: {
    billingDb: { type: "mongo", options: { uri: Deno.env.get("MONGO_URI") } },
  },
  apps: {
    billing: {
      definition: billingApp,
      uses: [{ slot: "database", resourceName: "billingDb" }],
      behaviors: { calculateDiscount: (order) => order.total * 0.1 },
    },
    invoicing: {
      definition: invoicingApp,
      uses: [{ slot: "database", resourceName: "billingDb" }],
    },
  },
});
```

Every entry under `apps` is a `defineZanixApp()` manifest
(`ZanixAppBootstrapOptions` — `{definition, server?, uses?, behaviors?}`). All
named apps (plus `admin`, when enabled — `core-admin-apis`) resolve together as
one batch, so two apps declaring a dependency on the same root resource share a
single instance instead of each constructing their own.

- **`SetupOptions.resources`** declares root-level resources any named app can
  bind against.
- **`uses`** resolves one of an app's own manifest `dependencies` slots to a
  concrete `resources` entry — the binding's `appName` is that entry's own
  `apps` key, never repeated in `uses` itself.
- **`behaviors`** overrides a pure function/strategy slot the app's own manifest
  declared (see `@zanix/app`'s own `app-behaviors-and-overrides`) — each key
  must be a `behaviors` name that app actually declared; `Zanix.start()` throws
  before constructing anything if it isn't, or if `apps` never declares that app
  at all.

An entry with no `server` still registers (mount, jobs, resources,
`setup`/`onStart`/`onStop`) but is never served over HTTP — useful for an app
that only needs background jobs or shared resources.

## `Zanix.setup()`: cross-cutting configuration

```ts
Zanix.setup({
  errors: {
    logThrottle: { threshold: 100, windowMs: 10 * 60_000 },
    uncaughtMonitor: { exitOnThreshold: true },
  },
  logger: { elastic: true },
  database: { seeders: false, triggersModel: "my-triggers" },
  notifications: { templatesBackend: "local" },
  dlq: { modelName: "my-dlq", encryptPayload: true },
});

await Zanix.start();
```

Wires error-handling config, logger configuration (including
Elasticsearch/OpenSearch-backed persistence), and non-secret
`@zanix/datamaster`/`@zanix/notifications` config that would otherwise mean
setting several env vars by hand.

| Option                                                                                                   | Sets / wires                                                                                                                                                                                                                          |
| -------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `errors.logThrottle`                                                                                     | `@zanix/server`'s `ErrorLogThrottle` (repeated-HTTP-status log noise) — no env var fallback, explicit options only.                                                                                                                   |
| `errors.uncaughtMonitor`                                                                                 | `@zanix/server`'s `UncaughtErrorMonitor` (process-wide uncaught-error/unhandled-rejection rate tracking feeding readiness — `threshold`/`windowMs`/`exitOnThreshold`), a separate concern from `logThrottle`, same shape, no env var fallback. |
| `logger.elastic`                                                                                         | Elasticsearch/OpenSearch-backed logging — `true` (or unset while `SEARCH_ENGINE` resolves to `elasticsearch`/`opensearch` — via `resolveSearchEngine()`, re-derived here rather than a blind env-var check; `meilisearch` deliberately does NOT trigger this) enables it; `false` opts out even then. |
| `logger.formatter` / `disableGlobalAssign`                                                               | Forwarded as-is to `@zanix/utils`'s `Logger`. No `storage.save` option here — construct your own `new Logger({storage:{save}})` directly for a fully custom save function.                                                            |
| `database.seeders` / `.triggersModel` / `.seedModel` / `.triggersPollInterval` / `.triggersChangeStream` | `DATABASE_SEEDERS` / `TRIGGERS_MODEL_NAME` / `SEED_MODEL_NAME` / `TRIGGERS_POLL_INTERVAL` / `TRIGGERS_CHANGE_STREAM`.                                                                                                                 |
| `notifications.templatesBackend` / `.templatesModel`                                                     | `TEMPLATES_BACKEND` / `TEMPLATES_MODEL_NAME`.                                                                                                                                                                                        |
| `dlq.modelName` / `.encryptPayload` / `.defaultLeaseMs`                                                  | `DLQ_MODEL_NAME` / `DLQ_ENCRYPT_PAYLOAD` / `DLQ_DEFAULT_LEASE_MS`.                                                                                                                                                                    |
| `assets.*`                                                                                               | See below — the one option that also constructs real infrastructure.                                                                                                                                                                  |

**An already-set env var always wins** — every option here only sets its env var
when not already present (an empty string counts as not present). The deployment
platform/container's own config is the authority; these options are just the
app-level default for when nothing else specified a value. To force a value
regardless of environment, set it directly via `Deno.env.set()` instead.

## `assets`: the one option that builds real infrastructure

```ts
import { defineSpaceApp } from "jsr:@zanix/space@[version]";

Zanix.setup({
  assets: { s3Bucket: "prod-assets", localDir: "./local-assets" },
});

const app = defineSpaceApp({
  name: "storefront",
  assetsApi: { service: Zanix.getAssetsService()! },
});

await Zanix.start({
  apps: { storefront: { definition: app, server: { ssr: {} } } },
});
```

Unlike `database`/`notifications`/`dlq` (env vars only), `assets` **also
constructs real infrastructure** and self-registers it — the same shape
`logger.elastic` already has (`new Logger()` self-registers globally;
`import logger from '@zanix/logger'` reads it back). Builds `S3ObjectStorage`
(renamed from `SeaweedFSObjectStorage` — it was never actually
SeaweedFS-specific, see `datamaster/CHANGELOG.md`) (+ an optional local
fallback/migration when `localDir` is given) and `MongoFileRepository` (adapted
via `@zanix/space/assets-api`'s `createAssetRepositoryOverFiles`), wires them
into a real `AssetService`, and registers it — read it back with
`Zanix.getAssetsService()`.

There is no separate "enabled" flag — passing the `assets` block at all (even
`{}`) is the activation signal; every field, including `localDir`, is optional,
each sets its own env var only when not already present, same precedence as
every other option. Omit `assets` entirely and nothing is constructed, at zero
cost.

**Real trade-off worth knowing**: omitting `localDir` means the constructed
`AssetService` talks to S3 directly with no local fallback at all — reads/writes
genuinely fail if S3 is unreachable, rather than silently degrading to local
disk.

## Checklist before bootstrapping or configuring a project

- [ ] Is `Zanix.setup()` called before `Zanix.start()`/`Zanix.startWorker()` —
      not after, where its env-var-setting options would have no effect?
- [ ] Does a named app sharing a resource with another actually declare `uses`
      against the same `resources` key, rather than each constructing its own
      connector?
- [ ] If both a named app's `server.ssr` and `main`'s should share one listener,
      is an identical explicit `port` passed to both?
- [ ] If `assets` is configured, is `localDir` deliberately included or omitted
      based on whether S3-unreachable-fails-hard is the intended behavior for
      this deployment?

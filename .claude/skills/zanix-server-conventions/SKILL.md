---
name: zanix-server-conventions
description: Real-world conventions and idiomatic patterns for building/reviewing backend microservices on @zanix/server (handlers, interactors, providers/repositories, middlewares, sockets, jobs) — distilled from actual production Zanix apps, not just the library's own docs. Use it when writing, reviewing, or debugging code in a Zanix backend service.
---

This skill is about writing and reviewing **application code that consumes `@zanix/server`**, not
about maintaining the library itself (see `jsdoc-jsr-audit` / `docs-readme-audit` for that). The
patterns below were confirmed against real production Zanix microservices, cross-checked against
`@zanix/server`'s own source — not guessed from the library's docs alone. Where something is a
convention (how real apps happen to structure things) vs. a hard framework rule (verified in
`@zanix/server`'s source), it's called out explicitly.

Applying these patterns to a new handler/interactor/provider, or adjusting an existing one, is
itself a feature addition or modification in the Zanix service's own codebase — run it through
`feature-completeness-conventions`' gate (tests for the new/changed behavior, docs updated if the
service documents its own endpoints/jobs, JSDoc accurate on any exported symbol) rather than
treating "it follows the conventions below" as the finish line.

## Entry point: `@zanix/core`, not raw `bootstrapServers`

Real apps don't call `bootstrapServers` from `@zanix/server` directly. `main.ts` is almost always
just:

```ts
import Zanix from '@zanix/core'

Zanix.start({
  server: {
    rest: { globalPrefix: 'auth' },
  },
})
```

`@zanix/core`'s `Zanix.start()` wraps `bootstrapServers` internally (plus config loading and
internal/admin server wiring) — treat `@zanix/server`'s `bootstrapServers`/`webServerManager` as the
underlying mechanism, not something app code typically calls by hand. If a project also has a
`worker.ts`, it's usually just `import '@zanix/asyncmq/worker'` — a separate process entrypoint for
the queue/cron worker, run independently from the HTTP server process.

## Project structure actually observed

```
src/
  server/
    handlers/            *.handler.ts — Controllers, Resolvers, Sockets
      rtos/               RTO classes (BaseRTO subclasses) used for validation
    interactors/          *.service.ts — business logic (often named *.service.ts, not
                          *.interactor.ts; the class still extends ZanixInteractor)
    providers/            *.defs.ts / standalone providers not tied to a single model
    repositories/
      <name>/
        entity.provider.ts   the ZanixProvider subclass for that model
        model.defs.ts        model/schema registration (via a companion ORM package)
        seeders/
          main.ts, seeders.dev.ts, seeders.prod.ts
    jobs/                 *.defs.ts — registerCronJob definitions
  shared/
    middlewares/          *.defs.ts — app-level guard/pipe/interceptor definitions
  utils/
```

`@zanix/server` itself doesn't scan the filesystem by suffix — only `@zanix/core`'s bootstrap does,
and it tolerates real-world naming drift (`*.service.ts` for interactors works fine as long as the
file gets imported transitively, e.g. from a handler). Don't "fix" a project's naming to match
`@zanix/server`'s own README table unless it's actually breaking auto-discovery.

## What `@zanix/cli` actually scaffolds (CLI scaffolding convention)

Running `zanix new server`/`app-server` creates the skeleton below, but as of this writing **every
server-specific file it writes is empty (0 bytes)** — verified directly (`find` + byte count on a
freshly scaffolded project), not assumed. This is a real, independently tracked bug in the
underlying template-content mechanism (`@zanix/utils`'s tree builder fetches each file's boilerplate
live from the owning library's published JSR `src/templates/` directory; `@zanix/core`'s is empty
and `@zanix/server`'s doesn't exist yet). Treat the scaffolded tree as **where files go**, not as a
source of example content — every pattern in this skill comes from real, hand-written production
code, not from what the CLI currently outputs.

```
src/server/
  connectors/example.connector.ts    (empty)
  handlers/example.handler.ts        (empty)
    rtos/example.rto.ts              (empty)
  interactors/service.interactor.ts  (empty)
  jobs/job.defs.ts                   (empty)
  repositories/
    model.defs.ts                    (empty, FLAT — one file, not one per model)
    seeders/seeder.ts                (empty, a single file)
shared/middlewares/{pipe,interceptor}.defs.ts   (empty)
zanix/config.ts                                  (empty)
```

Concrete divergences from "Project structure actually observed" above, and why (investigated, not
assumed — the CLI isn't automatically "the truth" just because it's newer):

- **`connectors/` is scaffolded but real projects examined don't use it.** Every `@zanix/datamaster`-backed
  project sampled gets its connector (`ZanixMongoConnector`) from the companion library, not a
  hand-written one — matching this skill's own Providers/Repositories section below. Delete the
  folder rather than filling in `example.connector.ts` if the project only uses library-supplied
  connectors.
- **`repositories/` starts flat; real projects restructure to nested-per-model almost immediately.**
  The CLI's single `model.defs.ts` is a reasonable *empty-project* starting point, but every real
  repository sampled uses `repositories/<name>/{entity.provider.ts,model.defs.ts,seeders/}` (as
  documented below) as soon as there's more than one model. Restructure into that shape the moment
  a second repository is needed, rather than growing the flat file.
- **Seeders scaffold a single `seeder.ts`, not the real three-file trio.** Every real repository's
  seeders are `main.ts` (re-exports `defineSeeders(seedersProd, seedersDev)`) + `seeders.dev.ts` +
  `seeders.prod.ts` — verified byte-identical across every repository sampled, not the CLI
  whole-project baseline's single-file shape. `zanix generate seeder <name>` (a newer, separate
  `@zanix/cli` command) scaffolds this real trio directly into an existing project — prefer it over
  hand-copying the pattern, and prefer it over the whole-project tree's `seeder.ts` example.
- **`main.ts`/`worker.ts` are not scaffolded at all.** The Entry point section above
  (`import Zanix from '@zanix/core'; Zanix.start(...)`, optionally a separate `worker.ts` for
  `@zanix/asyncmq/worker`) has to be hand-written after `zanix new` — there's no template for it yet.
- **`handlers/rtos/validations/IsObjectID.ts` (a custom Mongo ObjectId validator) isn't scaffolded
  either**, but showed up hand-written, nearly identically, in every real project sampled that
  validates an ObjectId in an RTO. Expect to write this by hand today; it's a candidate for a future
  `zanix generate` addition, not an existing one.

## Handlers

Controllers stack a class decorator with `Interactor` injection, then per-method HTTP decorators
with RTOs:

```ts
import { Controller, Get, Post, type HandlerContext, ZanixController } from '@zanix/server'
import { AuthTokenValidation, RateLimitGuard } from '@zanix/auth'
import { UserService } from '../interactors/users.service.ts'
import { RequestRegisterRTO, UserRegisterRTO } from './rtos/user-settings.ts'

@Controller({ prefix: 'user', Interactor: UserService })
export class UsersController extends ZanixController<UserService> {
  @Get('access-request/:email', { Params: RequestRegisterRTO })
  @RateLimitGuard({ anonymousLimit: 1 })
  public requestRegister(ctx: HandlerContext<{ params: RequestRegisterRTO }>) {
    return this.interactor.requestRegister(ctx.payload.params.email)
  }

  @Post('register', { Body: UserRegisterRTO })
  @AuthTokenValidation({ permissions: ['user:write'] })
  public register(ctx: HandlerContext<{ body: UserRegisterRTO }>) {
    return this.interactor.registerNewUser(ctx.payload.body)
  }
}
```

- **`@AuthTokenValidation`/`@RateLimitGuard` (from `@zanix/auth`) are the norm — bare `@Guard` from
  `@zanix/server` almost never appears directly in handler code.** If a project needs a new
  cross-cutting concern (auth, rate limiting, tenant scoping, etc.), the right move is building one
  reusable decorator on top of `defineMiddlewareDecorator` (see `@zanix/server`'s Middlewares guide)
  and reusing it everywhere — not scattering bare `@Guard(fn)` calls per handler.
- `@Post(rto)`/`@Get(rto)` etc. with no path string is common — the method name becomes the route
  (e.g. a method named `logout` decorated with `@Post({ Body: TokenRTO })` registers `POST /logout`).

## WebSocket handlers

```ts
import { Socket, ZanixWebSocket } from '@zanix/server'
import { AuthTokenValidation } from '@zanix/auth'

@Socket({ route: 'overview', rto: CashPositionRTO })
@AuthTokenValidation()
export class OverviewSocket extends ZanixWebSocket {
  get #connectionId() {
    return this.context.session?.id
  }

  protected override onopen() {
    if (this.#connectionId) this.registry.set(this.#connectionId, this)
  }

  protected override onclose() {
    if (this.#connectionId) this.registry.delete(this.#connectionId)
  }

  public push(data: object) {
    this.socket.send(JSON.stringify(data))
  }
}
```

- `this.registry` (a generic key-value store, also available on providers/interactors) is the real
  pattern for tracking live connections by a custom id, so another part of the app — a job, an
  interactor, a REST handler — can look one up later and push to it (`this.registry.get<T>(id)`).
- `this.socket.send()` is how you push data outside the request/response cycle of `onmessage`
  (background updates, notifications triggered elsewhere).
- **Framework rule, verified in `@zanix/server`'s source**: `@Guard`/`@Pipe`/`@Interceptor` only
  take effect on a `@Socket` class at the **class** level (as shown above). Putting one directly on
  `onmessage`/`onopen`/etc. has no effect — a socket has exactly one route (the connection/upgrade),
  not one per lifecycle method, so there's no per-method route for it to attach to. Don't try to
  "fix" this in application code; it's a framework-level limitation, not a bug in the app.

## Interactors / Services

The single dependency declared via `@Interactor({ Connector, Provider })` is accessed through
`this.connector`/`this.provider`. For anything beyond that one dependency — which is the common
case — use the dynamic getters:

```ts
export class AuthService extends ZanixInteractor {
  public async login(email: string, password: string) {
    const users = this.providers.get(UsersRepository)
    const roles = this.providers.get(RolesRepository)
    // ...
    const password = this.interactors.get(PasswordService)
    return password.validate(/* ... */)
  }
}
```

- `this.providers.get(X)` / `this.connectors.get(X)` reach ANY registered provider/connector by
  class, not just the one declared on the decorator — this is the dominant real pattern, used to
  pull in several repositories from a single service method.
- `this.interactors.get(OtherService)` calls another interactor/service directly (a
  self-reference — `this.interactors.get(Self)` from inside one of its own methods — resolves to the
  same instance rather than recursing; it shows up as a way to guarantee the request-scoped instance
  is used).
- **Framework rule, verified in `@zanix/server`'s source**: don't pass a class that extends one of
  the built-in "core" base classes (`ZanixDatabaseConnector`, `ZanixCacheConnector`,
  `ZanixKVConnector`, `ZanixAsyncmqConnector`, `ZanixCacheProvider`, `ZanixWorkerProvider`,
  `ZanixAsyncMQProvider`) as `@Interactor`'s `Connector`/`Provider` option — it throws an
  `InternalError` telling you to access it via the matching named getter instead
  (`this.database`, `this.cache`, etc., without declaring it on the decorator at all). Only
  **custom** connectors/providers (extending `ZanixConnector`/`ZanixProvider` directly, or a
  repository-style `ZanixProvider` subclass) belong in the decorator options.

## Providers / Repositories

The idiomatic pattern for a "repository" provider is a named connector slot, with the concrete
connector coming from a companion package (e.g. `@zanix/datamaster` for databases), not from
`@zanix/server` itself:

```ts
import type { ZanixMongoConnector } from '@zanix/datamaster'
import { Provider, ZanixProvider } from '@zanix/server'

@Provider()
export class UsersRepository extends ZanixProvider<{ database: ZanixMongoConnector }> {
  public findByEmail(email: string) {
    return this.database.getModel('User').findOne({ email })
  }

  // Self-reference to force resolving the request-scoped instance:
  public findByEmailAndValidateAccess(email: string) {
    return this.providers.get(UsersRepository).findByEmail(email)
  }
}
```

Model/schema registration for a repository typically lives in a sibling `model.defs.ts`, wired
through whichever ORM/ODM companion package the project uses (e.g. `@zanix/datamaster`'s
`registerModel(...)`), including per-repository seeders (`seeders/main.ts` + environment-specific
`seeders.dev.ts`/`seeders.prod.ts`).

## Jobs / queues

Cron and queue job registration comes from `@zanix/asyncmq`, not `@zanix/server` (the
`BackoffOptions`/`ScheduleOptions`/etc. types `@zanix/server` exports are just shared typings that
package consumes — `@zanix/server` has no job-registration API of its own):

```ts
import { registerCronJob } from '@zanix/asyncmq'

registerCronJob({
  isActive: true,
  name: 'generate-reports',
  processingQueue: 'soft',
  schedule: '0 0 */1 * * ?', // every hour
  handler: async function () {
    const repository = this.providers.get(ReportsRepository)
    // ...
  },
})
```

The job handler's `this` context exposes the same `this.providers.get(...)`/`this.connectors.get(...)`
pattern as interactors and providers — it's the one dependency-access mechanism reused across the
whole ecosystem, not something specific to handlers.

## Review checklist

When reviewing a PR (or writing new code) in a Zanix backend service, check for:

- [ ] Is a bare `@Guard`/`@Pipe`/`@Interceptor` being added directly to a handler where an existing
      org-level decorator (auth, rate limit, etc.) should be reused or extended instead?
- [ ] Does an interactor/provider only use its single declared `Connector`/`Provider` when it
      actually needs several — should it use `this.providers.get(X)`/`this.connectors.get(X)`
      instead of adding more single-slot dependencies?
- [ ] Is a *core* connector/provider (one extending a built-in base class) being passed into
      `@Interactor`'s options? That throws — it should be accessed via the named getter instead.
- [ ] Is `@Guard`/`@Pipe`/`@Interceptor` being applied to a Socket's `onmessage`/`onopen`/etc.
      instead of the class? It silently won't do anything.
- [ ] Does a WebSocket handler that needs to push data outside of `onmessage` actually track its
      connection via `this.registry` (or does it lose the ability to be reached from elsewhere)?
- [ ] Are RTOs (`Body`/`Params`/`Search`) used for request validation, rather than manual
      validation inside the handler body?
- [ ] For new/changed handlers, interactors, providers, or jobs: does the change pass the
      `feature-completeness-conventions` gate (tests covering the new/changed behavior, docs/JSDoc
      updated if stale, no lingering assertion of the old behavior)?

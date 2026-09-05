---
name: zanix-server-conventions
description: Real-world conventions and idiomatic patterns for building/reviewing backend microservices on @zanix/server (handlers, interactors, providers/repositories, middlewares, sockets, jobs) — distilled from actual production Zanix apps, not just the library's own docs. Use it when writing, reviewing, or debugging code in a Zanix backend service.
---

This skill is about writing and reviewing **application code that consumes
`@zanix/server`**, not about maintaining the library itself (see
`jsdoc-jsr-audit` / `docs-readme-audit` for that). The patterns below were
confirmed against real production Zanix microservices, cross-checked against
`@zanix/server`'s own source — not guessed from the library's docs alone. Where
something is a convention (how real apps happen to structure things) vs. a hard
framework rule (verified in `@zanix/server`'s source), it's called out
explicitly.

Applying these patterns to a new handler/interactor/provider, or adjusting an
existing one, is itself a feature addition or modification in the Zanix
service's own codebase — run it through `feature-completeness-conventions`' gate
(tests for the new/changed behavior, docs updated if the service documents its
own endpoints/jobs, JSDoc accurate on any exported symbol) rather than treating
"it follows the conventions below" as the finish line.

## Golden rule (token savings)

- The review checklist at the end is the fast path for reviewing existing
  code — run through it directly against the diff instead of re-reading every
  section's full example code first.
- When writing new code, copy the shape from this skill's own example for the
  matching artifact (handler/interactor/provider/socket/job) and adapt it,
  rather than re-deriving the pattern from `@zanix/server`'s source each time.

## Entry point: `@zanix/core`, not raw `bootstrapServers`

Real apps don't call `bootstrapServers` from `@zanix/server` directly. `main.ts`
is almost always just:

```ts
import Zanix from "@zanix/core";

Zanix.start({
  server: {
    rest: { globalPrefix: "auth" },
  },
});
```

`@zanix/core`'s `Zanix.start()` wraps `bootstrapServers` internally (plus config
loading and internal/admin server wiring) — treat `@zanix/server`'s
`bootstrapServers`/`webServerManager` as the underlying mechanism, not something
app code typically calls by hand. If a project also has a `worker.ts`, it's
usually just `import '@zanix/asyncmq/worker'` — a separate process entrypoint
for the queue/cron worker, run independently from the HTTP server process.

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

`@zanix/server` itself doesn't scan the filesystem by suffix — only
`@zanix/core`'s bootstrap does, and it tolerates real-world naming drift
(`*.service.ts` for interactors works fine as long as the file gets imported
transitively, e.g. from a handler). Don't "fix" a project's naming to match
`@zanix/server`'s own README table unless it's actually breaking auto-discovery.

## Handlers

Controllers stack a class decorator with `Interactor` injection, then per-method
HTTP decorators with RTOs:

```ts
import {
  Controller,
  Get,
  type HandlerContext,
  Post,
  ZanixController,
} from "@zanix/server";
import { AuthTokenValidation, RateLimitGuard } from "@zanix/auth";
import { UserService } from "../interactors/users.service.ts";
import { RequestRegisterRTO, UserRegisterRTO } from "./rtos/user-settings.ts";

@Controller({ prefix: "user", Interactor: UserService })
export class UsersController extends ZanixController<UserService> {
  @Get("access-request/:email", { Params: RequestRegisterRTO })
  @RateLimitGuard({ anonymousLimit: 1 })
  public requestRegister(ctx: HandlerContext<{ params: RequestRegisterRTO }>) {
    return this.interactor.requestRegister(ctx.payload.params.email);
  }

  @Post("register", { Body: UserRegisterRTO })
  @AuthTokenValidation({ permissions: ["user:write"] })
  public register(ctx: HandlerContext<{ body: UserRegisterRTO }>) {
    return this.interactor.registerNewUser(ctx.payload.body);
  }
}
```

- **`@AuthTokenValidation`/`@RateLimitGuard` (from `@zanix/auth`) are the norm —
  bare `@Guard` from `@zanix/server` almost never appears directly in handler
  code.** If a project needs a new cross-cutting concern (auth, rate limiting,
  tenant scoping, etc.), the right move is building one reusable decorator on
  top of `defineMiddlewareDecorator` (see `@zanix/server`'s Middlewares guide)
  and reusing it everywhere — not scattering bare `@Guard(fn)` calls per
  handler.
- `@Post(rto)`/`@Get(rto)` etc. with no path string is common — the method name
  becomes the route (e.g. a method named `logout` decorated with
  `@Post({ Body: TokenRTO })` registers `POST /logout`).

### Unit-testing a Controller method

`@zanix/server` ships no testing helpers of its own for this — confirmed:
its `deno.jsonc` `exports` is just `{".": "./mod.ts"}`, no `/testing`
subpath (unlike `@zanix/space`'s `mockHandlerContext`/`mockPageContext`).
Two real options, verified against `ZanixController`/`HandlerBaseClass`'s
own source:

- **Prefer testing the interactor directly, not the controller.** A
  controller method is usually a thin `return this.interactor.xxx(...)`
  dispatch — test `UserService`'s own methods with `new UserService()` and
  mocked providers, which needs no `HandlerContext` at all. This is the
  simpler, more common real pattern and covers the actual business logic.
- **If the controller method itself has real logic worth covering**
  (response shaping, reading multiple `ctx.payload` fields), construct only
  the real `HandlerContext` fields the method under test actually reads
  (`{ id: 'test', ...} as HandlerContext` — a full interface satisfies
  `req`/`url`/`payload`/`cookies`/`locals`, but a partial cast is fine for a
  method that doesn't touch the rest) and override `this.interactor` on the
  constructed instance directly, since it's a real getter that resolves
  through `ProgramModule.targets.getInteractor(...)` — never wired up in a
  bare unit test:
  ```ts
  const instance = new UsersController({ id: "test" } as HandlerContext);
  Object.defineProperty(instance, "interactor", { value: mockUserService });
  ```
  Constructing the class itself (`new UsersController(ctx)`) works fine
  outside the framework's own DI container — `@Controller`'s decorator sets
  the class's `ZANIX_PROPS` at class-definition time (module import), not at
  instantiation — it's specifically resolving `this.interactor` through
  `ProgramModule.targets` that needs the override above.

  **This is a permanent, regression-guarded claim about `@zanix/server`
  itself, not just a technique that happened to work once**:
  `server/src/@tests/unit/infra/handlers/interactor-override.test.ts` (added
  by `server-builder`, not by this skill) asserts all of the above directly
  against real source — including that `ProgramModule.targets.getInteractor`
  is called **zero** times when the override is used (a spy rigged to throw
  proves DI resolution is genuinely bypassed, not just coexisting with it) —
  and was confirmed to actually catch a regression (a temporary
  `Object.freeze(this)` in `TargetBaseClass`'s constructor made the test fail
  exactly as expected, reverted afterward). The precise mechanism, for
  anyone who wants it: `@Controller`'s decorator wiring is
  `defineControllerDecorator` → `ProgramModule.targets.defineTarget` →
  `toBeInstantiated`, fixing `ZANIX_PROPS` on the prototype with no DI
  container involved. If this pattern ever breaks, that test — not this
  skill's prose — is the first place to look.

## WebSocket handlers

```ts
import { Socket, ZanixWebSocket } from "@zanix/server";
import { AuthTokenValidation } from "@zanix/auth";

@Socket({ route: "overview", rto: CashPositionRTO })
@AuthTokenValidation()
export class OverviewSocket extends ZanixWebSocket {
  get #connectionId() {
    return this.context.session?.id;
  }

  protected override onopen() {
    if (this.#connectionId) this.registry.set(this.#connectionId, this);
  }

  protected override onclose() {
    if (this.#connectionId) this.registry.delete(this.#connectionId);
  }

  public push(data: object) {
    this.socket.send(JSON.stringify(data));
  }
}
```

- `this.registry` (a generic key-value store, also available on
  providers/interactors) is the real pattern for tracking live connections by a
  custom id, so another part of the app — a job, an interactor, a REST handler —
  can look one up later and push to it (`this.registry.get<T>(id)`).
- `this.socket.send()` is how you push data outside the request/response cycle
  of `onmessage` (background updates, notifications triggered elsewhere).
- **Framework rule, verified in `@zanix/server`'s source**:
  `@Guard`/`@Pipe`/`@Interceptor` only take effect on a `@Socket` class at the
  **class** level (as shown above). Putting one directly on
  `onmessage`/`onopen`/etc. has no effect — a socket has exactly one route (the
  connection/upgrade), not one per lifecycle method, so there's no per-method
  route for it to attach to. Don't try to "fix" this in application code; it's a
  framework-level limitation, not a bug in the app.

## Interactors / Services

`@Interactor()` takes no single-slot `Connector`/`Provider` option (that was
removed in `@zanix/server` v3.0.0) — every dependency, whatever the count, is
reached through the dynamic getters:

```ts
export class AuthService extends ZanixInteractor {
  public async login(email: string, password: string) {
    const users = this.providers.get(UsersRepository);
    const roles = this.providers.get(RolesRepository);
    // ...
    const password = this.interactors.get(PasswordService);
    return password.validate(); /* ... */
  }
}
```

- `this.providers.get(X)` / `this.connectors.get(X)` reach ANY registered
  provider/connector by class — including a built-in "core" base class
  (`ZanixDatabaseConnector`, `ZanixCacheConnector`, `ZanixKVConnector`,
  `ZanixAsyncmqConnector`, `ZanixCacheProvider`, `ZanixWorkerProvider`,
  `ZanixAsyncMQProvider`) via its matching named getter instead
  (`this.database`, `this.cache`, etc.) — the same one mechanism used to pull
  in several repositories from a single service method.
- `this.interactors.get(OtherService)` calls another interactor/service directly
  (a self-reference — `this.interactors.get(Self)` from inside one of its own
  methods — resolves to the same instance rather than recursing; it shows up as
  a way to guarantee the request-scoped instance is used).

## Providers / Repositories

The idiomatic pattern for a "repository" provider is a named connector slot,
with the concrete connector coming from a companion package (e.g.
`@zanix/datamaster` for databases), not from `@zanix/server` itself:

```ts
import type { ZanixMongoConnector } from "@zanix/datamaster";
import { Provider, ZanixProvider } from "@zanix/server";

@Provider()
export class UsersRepository
  extends ZanixProvider<{ database: ZanixMongoConnector }> {
  public findByEmail(email: string) {
    return this.database.getModel("User").findOne({ email });
  }

  // Self-reference to force resolving the request-scoped instance:
  public findByEmailAndValidateAccess(email: string) {
    return this.providers.get(UsersRepository).findByEmail(email);
  }
}
```

Model/schema registration for a repository typically lives in a sibling
`model.defs.ts`, wired through whichever ORM/ODM companion package the project
uses (e.g. `@zanix/datamaster`'s `registerModel(...)`), including per-repository
seeders (`seeders/main.ts` + environment-specific
`seeders.dev.ts`/`seeders.prod.ts`).

### Sharing a Provider between REST and Space pages (`space-server`)

A `space-server` project that deliberately binds a Space page's own Interactor
to the SAME Provider/Repository class its REST API already registers (real,
tested business logic reused instead of duplicated — as opposed to a
Space-only Interactor that calls the REST API over HTTP instead) hits a real,
confirmed module-identity crash unless the shared class opts into a **custom
slot**:

```ts
@Provider('authRepository') // <- a plain string, not one of the 5 core slots
export class AuthRepository extends ZanixProvider {
  /* ... */
}
```

**Why this is needed, mechanically**: the native `zanix space dev`/production
process loads a project-local Provider class directly, once. But a Space page
reached through `@zanix/space`'s Vite-backed SSR pipeline (`zanix space dev`
only — see `space-routing-and-rendering`'s own note) RE-EVALUATES that same
source file as a SECOND, independent class object — same source, same name,
different reference. `NATIVE_RUNTIME_MODULES` (`@zanix/space`) fixes this
exact problem for `@zanix/*` PACKAGES (`@zanix/auth`, `@zanix/datamaster`,
...), but structurally cannot reach a project's OWN class at all. Without a
shared `slot`, `this.providers.get(AuthRepository)` called from the
SSR-side evaluation can never find what the native evaluation registered:
`[BaseInstancesContainer]: Target is not a constructor` — `INVALID_INSTANCE`,
a real, reported production failure (a real POST to a Space login page's own
`action`, calling into a shared `AuthService`/`AuthRepository`).

A `slot` (core or custom) makes `this.providers.get(TheClass)` and
`this.providers.get('theSlot')` resolve the IDENTICAL cached singleton
regardless of which evaluation's class reference asks — no call site needs to
change, only the `@Provider(...)` decorator itself. Confirmed (a real
constructor-call counter, both resolution orders) to share exactly one
constructed instance, never two independently-live ones — safe for a typical
stateless-ish Provider/Repository (state in an external DB/cache, not
class-level static fields). **Requires `@zanix/server >= 4.2.1`** — a custom
`slot` string was silently a no-op before that (resolved purely by class
reference, the same as omitting `slot` entirely).

Only needed for a Provider/Repository a Space page's own Interactor reaches
directly by class reference. A REST-only provider, or one only ever looked up
by its already-core-slotted class (`this.database`, `this.cache`, ...), never
crosses this boundary and needs no custom `slot` at all.

### External-service connectors: `RestClient`/`GraphQLClient`, never raw `fetch`

The generic (non-database) sibling of the pattern above: a `Connector`
extending `@zanix/server`'s own `RestClient` (REST) or `GraphQLClient`
(GraphQL, itself `extends RestClient`) — `zanix generate connector <name>`
scaffolds this exact shape. Reaching for a bare `fetch()` call inside a
handler/interactor/provider to talk to an external HTTP/GraphQL API is a
real, confirmed anti-pattern: it silently drops what the base class already
gives for free (default headers, conditional `ETag` caching, structured
`RestClientError`s, `GraphQLClient`'s spec-standard `introspect()`), and
scatters what should be one reusable connector instance across every call
site that needs it. A `Connector` is registered like any other
provider/repository and reached the same way (`this.providers.get(X)` or a
named getter for a core slot) — there's no reason for outbound HTTP/GraphQL
logic to look structurally different from the database-repository pattern
above.

## Jobs / queues

Cron and queue job registration comes from `@zanix/asyncmq/jobs`, not
`@zanix/server` (the `BackoffOptions`/`ScheduleOptions`/etc. types
`@zanix/server` exports are just shared typings that package consumes —
`@zanix/server` has no job-registration API of its own). `registerJob`/
`registerCronJob` live on the `/jobs` subpath, not the bare `@zanix/asyncmq`
entry point — that root import additionally bundles the RabbitMQ connector/
providers/subscribers, which a project that only wants to declare jobs
doesn't need to pull in:

```ts
import { registerCronJob } from "@zanix/asyncmq/jobs";

registerCronJob({
  isActive: true,
  name: "generate-reports",
  processingQueue: "soft",
  schedule: "0 0 */1 * * ?", // every hour
  handler: async function () {
    const repository = this.providers.get(ReportsRepository);
    // ...
  },
});
```

The job handler's `this` context exposes the same
`this.providers.get(...)`/`this.connectors.get(...)` pattern as interactors and
providers — it's the one dependency-access mechanism reused across the whole
ecosystem, not something specific to handlers.

## Review checklist

When reviewing a PR (or writing new code) in a Zanix backend service, check for:

- [ ] Is a bare `@Guard`/`@Pipe`/`@Interceptor` being added directly to a
      handler where an existing org-level decorator (auth, rate limit, etc.)
      should be reused or extended instead?
- [ ] Is a _core_ connector/provider (one extending a built-in base class)
      being accessed via its named getter (`this.database`, `this.cache`,
      etc.) rather than `this.providers.get(X)`/`this.connectors.get(X)`,
      where the named getter is the more idiomatic form?
- [ ] Is `@Guard`/`@Pipe`/`@Interceptor` being applied to a Socket's
      `onmessage`/`onopen`/etc. instead of the class? It silently won't do
      anything.
- [ ] Does a WebSocket handler that needs to push data outside of `onmessage`
      actually track its connection via `this.registry` (or does it lose the
      ability to be reached from elsewhere)?
- [ ] Are RTOs (`Body`/`Params`/`Search`) used for request validation, rather
      than manual validation inside the handler body?
- [ ] Does a handler/interactor/provider call a raw `fetch()` against an
      external HTTP/GraphQL API instead of a `Connector` extending
      `RestClient`/`GraphQLClient` (`zanix generate connector`)? See
      "External-service connectors" above.
- [ ] In a `space-server` project: does a Space page's own Interactor reach a
      Provider/Repository ALSO registered by the REST API, by class reference,
      with no custom `slot`? See "Sharing a Provider between REST and Space
      pages" above — a real crash, not a style nit.
- [ ] For new/changed handlers, interactors, providers, or jobs: does the change
      pass the `feature-completeness-conventions` gate (tests covering the
      new/changed behavior, docs/JSDoc updated if stale, no lingering assertion
      of the old behavior)?

## Out of scope — do not do these

- Extending or modifying `@zanix/server` itself (Application/Runtime/
  WebServerManager, route registration, the core-slot registry) —
  `zanix-server-internals`'s job; this skill only covers writing application
  code that consumes the library as-is.
- Auth mechanics behind `@AuthTokenValidation`/`@RateLimitGuard`/`@IpAllowlistGuard`
  (token verification, session rotation, rate-limit plans) — the `auth-*`
  skills own that; this skill only says to reach for the decorator instead of
  a bare `@Guard`.
- Queue/cron registration semantics (`registerCronJob`, job vs. task routing,
  DLQ/retry behavior) beyond the one example shown — that's
  `asyncmq-worker-and-tasks`/`asyncmq-scheduling-and-cron`'s job; `@zanix/server`
  only supplies the shared option types those APIs consume.
- Database/ORM model registration, connector setup, or caching patterns behind
  `this.database`/`this.cache` — the `datamaster-*` skills own the connector
  side; this skill only shows how a repository provider reaches it.
- Deciding whether a new resource's HTTP surface belongs in the owning
  package (a local API) or in `@zanix/admin` as an aggregator —
  `zanix-local-api-vs-aggregator`'s job, not a server-conventions question.
- Test coverage completeness, README/JSDoc audits, and release mechanics for
  the *consuming* project — `feature-completeness-conventions` /
  `docs-readme-audit` / `release-management` own those; this skill only flags
  when the gate applies, not what satisfies it.

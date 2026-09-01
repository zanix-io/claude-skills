# Zanix - Claude Skills

Reusable [Claude Code Skills](https://code.claude.com/docs/en/skills) for
working across the Zanix ecosystem — backend libraries and services today, with
room to grow into frontend and microapp projects as they adopt Claude Code.

## Structure

Skills are organized under `.claude/skills/` by how broadly they apply, so a new
skill's home is decided by the same question every time: _what's the smallest
scope this genuinely applies to?_

- **`universal/`** — stack-agnostic (or Deno/JSR-generic) engineering discipline
  that carries no Zanix-specific knowledge: test coverage, JSDoc, README/docs,
  the feature-completeness gate, and release/versioning mechanics. Useful even
  outside the Zanix ecosystem.
- **`ecosystem/`** — rules that cross more than one Zanix package: dependency
  direction between packages, the local-API-vs-aggregator ownership rule, and
  conventions for application code consuming `@zanix/server`.
- **`packages/<repo>/`** — knowledge specific to a single Zanix repo (its own
  internal architecture, generator conventions, testing setup) that would mean
  nothing to a sibling package. Covers all 12 Zanix ecosystem repos: `cli`,
  `server`, `datamaster`, `notifications`, `space-ui`, `asyncmq`, `space`,
  `auth`, `app`, `admin`, `core`, and `utils`.

## What's here

**`universal/`**

| Skill                              | Use it for                                                                                                                                                                                                                             |
| ---------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `complete-test-coverage`           | Auditing and closing real test-coverage gaps in any stack (Deno, Node, Python, Go...) without wasting tokens on dead/unreachable code.                                                                                                 |
| `jsdoc-jsr-audit`                  | Auditing existing JSDoc for accuracy (not just presence) and driving `deno doc --lint` to zero in Deno/JSR **packages**.                                                                                                               |
| `docs-readme-audit`                | Making a Deno/JSR **package**'s README, `docs/`, and CHANGELOG complete, coherent, and professional in one exhaustive pass.                                                                                                            |
| `feature-completeness-conventions` | Gating every new feature or change on tests, docs, and JSDoc all being complete and up to date — the umbrella convention tying the three audit skills above together.                                                                  |
| `release-management`               | CHANGELOG discipline, semver classification, and the branch → commit → push → tag sequence once a change is ready to ship.                                                                                                             |
| `documentation-voice`              | English-only, present-tense voice for JSDoc/docs/code — no session, plan, or phase/stage narrative, with a real `@deprecated` exception. Referenced by `jsdoc-jsr-audit`, `docs-readme-audit`, and `feature-completeness-conventions`. |
| `naming-and-structure-conventions` | Folder/file casing, test-file naming, constant-vs-behavior casing, the env-var `_ENV` constant-naming shape, and the `X-Znx-` HTTP header/cookie naming shape — the naming/structure rules uniform across every Zanix repo.             |
| `deno-lazy-dependency-pattern`     | Declaring a genuinely conditional/optional dependency in a published Deno/JSR package without `nodeModulesDir: "auto"` eagerly materializing its npm packages for every consumer — the `lazyFunction`/`lazyClass`/`lazyValue` pattern, the `import type` gotcha, and when to fix the source package's own export shape instead. |
| `deno-workspace-link-pitfalls`     | TEMPORARILY linking an unpublished sibling package via a raw relative-path override to test a fix before publish — `scopes` alias collisions, the exact prefix depth a `scopes` key needs, per-subpath override coverage, and unrelated sibling directories getting silently auto-linked. |

**`ecosystem/`**

| Skill                            | Use it for                                                                                                                                                                                        |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `zanix-dependency-direction`     | Deciding whether a new cross-package import is valid, or needs inversion via a registry — the tier hierarchy across `@zanix/server`/`datamaster`/`auth`/`notifications`/`asyncmq`/`admin`/`core`. |
| `zanix-local-api-vs-aggregator`  | Deciding whether a new HTTP controller belongs inside the domain package that owns the data, or in `@zanix/admin` as a cross-service aggregator.                                                  |
| `zanix-local-api-implementation` | The mechanics of actually building a local-API subpath once the rule above says it needs one — export structure, controller composition, the dependency-boundary test technique.                  |
| `zanix-server-conventions`       | Writing/reviewing **application** code on top of `@zanix/server` (handlers, interactors, providers, middlewares, sockets, jobs), grounded in real production Zanix microservices.                 |
| `zanix-envvar-conventions`       | Shaping a new env var correctly — the four-pattern taxonomy, the `X_MODE=a\|b\|c` + resolver precedent, and the `_ENV` constant-export convention.                                                |
| `zanix-observability-conventions`| How to log/throw/persist correctly across the ecosystem — log level and `noSave`, the 4 shared error classes, `message` vs. `userMessage`, connector-error sanitization, and when a new sensitive `X-Znx-` header/cookie needs registering for redaction. |
| `zanix-issue-reporting`          | When/how to file a real GitHub issue via `zanix report-issue` — the three-bucket rule for what gets reported vs. stays chat-only, repo detection, title/body shape.                               |
| `zanix-test-tier-conventions`    | Which of unit/integration/functional a new test belongs in — the real default every `zanix new`-scaffolded project gets, refined per-repo by a more specific testing skill when one exists.       |
| `zanix-remote-api-app-pattern`   | The layered pattern for a `@zanix/space` app that owns no data of its own and builds UI against a remote, typed Zanix API — resource descriptor, thin client, pages, presentation, auth — grounded in `@zanix/console`'s real Triggers/Templates slices. |
| `zanix-remote-api-app-e2e-validation` | Runbook for standing up a real, unmocked instance of the pattern above (business service + admin hub + console, three real processes) and driving it with `curl` through login and a full Triggers/Templates CRUD cycle — the only path that exercises the real network boundary every automated test in this area mocks. Applies the same way to a maintainer regression-proofing a library change and a consumer team validating their own trio before shipping. |
| `skill-and-agent-authoring`      | How this repo's own skills/agents get created and maintained — tier placement, the real symlink mechanism, the validation discipline, cross-agent boundary upkeep.                                |
| `zanix-agent-dispatch-discipline`| How to safely dispatch this repo's own agents — the standing gate that a named agent must actually run the work, the new-agent-type registration delay, concurrent same-repo dispatch verification, cross-session coordination. |

**`packages/cli/`**

| Skill                          | Use it for                                                                                                                            |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------- |
| `cli-command-architecture`     | `Commander`/`cliffy` command-tree wiring — `mountGroup`, error handling, the registration-function convention.                        |
| `cli-artifact-generators`      | Adding/changing a `zanix generate <artifact>` generator — module layout, the `plan<Name>` pattern, the parser/renderer split.         |
| `cli-scaffold-assembly`        | Adding a new project type/preset to `zanix new` — the Recipe/Assembler abstraction and the generator-vs-JSR-fetch ownership boundary. |
| `cli-dependency-compatibility` | Keeping generated code compiling against real, currently-published `@zanix/*` versions — pinned versions, Drift Watch, `--verify`.    |
| `cli-generator-testing`        | The three-tier test suite for generators, and how it complements Drift Watch/`--verify`.                                              |

**`packages/server/`**

| Skill                    | Use it for                                                                                                                                                                          |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `zanix-server-internals` | Extending `@zanix/server` itself — composition/activation layering, the core-slot registry, registration-function rules. Not for application code (see `zanix-server-conventions`). |

**`packages/datamaster/`**

| Skill                               | Use it for                                                                                                                                            |
| ----------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `datamaster-connector-registration` | Adding a new core connector slot — the `registerCoreConnectorSlot` pattern repeated identically across mongo/redis/qlru/sqlite/observability/storage. |
| `datamaster-database-and-models`    | `ZanixMongoConnector`, `registerModel`, seeders, multi-DB, multi-connector, pagination/search.                                                        |
| `datamaster-data-protection`        | mask/encrypt/hash + access strategies, versioned keys, key rotation.                                                                                  |
| `datamaster-triggers`               | Declarative reactive triggers on model lifecycle events, persisted/online-editable triggers, the local `/admin/triggers` API.                         |
| `datamaster-dlq`                    | The Mongo-backed Dead Letter Queue — lifecycle, leasing, payload protection.                                                                          |
| `datamaster-storage`                | Object storage (S3-compatible, e.g. SeaweedFS) and file-metadata registry, encryption at rest.                                                        |
| `datamaster-cache`                  | Redis/local-LRU connectors, the multi-layer cache provider, locking.                                                                                  |
| `datamaster-observability`          | Elasticsearch/OpenSearch connector and the `@zanix/logger` bridge.                                                                                    |

**`packages/notifications/`**

| Skill                                  | Use it for                                                                                                                                       |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `notifications-connectors`             | `SmtpClient`/`SmsClient`/`WhatsappClient`, pluggable provider adapters, native WhatsApp Business templates vs. this package's own templates.     |
| `notifications-provider`               | `NotifierProvider`'s send API, the `mail` trigger-action contract, background-worker queuing.                                                    |
| `notifications-templates`              | The Handlebars template system — built-in registries, compiled rendering, adding a base template.                                                |
| `notifications-template-inheritance`   | Templates that render through another template's content (`parent`) instead of owning their own — easy to conflate with the base-template guide. |
| `notifications-template-storage-modes` | Pure code vs. per-service/shared/remote database-backed template resolution (Modes A/B/C), and the local `/templates` CRUD API.                  |

**`packages/space-ui/`**

| Skill                            | Use it for                                                                                                                                                                 |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `space-ui-architecture`          | The seven seams every component must keep true, and the ownership map against the Application/`@zanix/space` — the governing rule for any new/changed component.           |
| `space-ui-foundation-primitives` | The shared toolkit (outside-click, Escape, focus-trap, live-region, roving-focus, positioning) — what's public, and the real-second-consumer extraction discipline.        |
| `space-ui-component-patterns`    | The build discipline (composed vs. reimplemented `data-space-ui`, render-prop pattern, controlled/uncontrolled shape) plus real React/Preact bugs already found and fixed. |
| `space-ui-styling`               | Headless-by-default architecture, `data-space-ui` hooks, `theme/` vs `shared/` templates, `--space-*` token composition.                                                   |
| `space-ui-icons`                 | The optional default icon catalog (`CatalogIcon`), `createCatalogIcon` for a project's own catalog, and a real `svgo` gotcha on multi-symbol sprites.                      |
| `space-ui-richtext`              | ICU rich-text tags, population via a typed sentinel, Markdown via a pure-AST parser, and the `resolveRichTextDocument` loader pattern.                                     |

**`packages/asyncmq/`**

| Skill                               | Use it for                                                                                                                                          |
| ----------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `asyncmq-connector-and-subscribers` | `ZanixRabbitMQConnector`, `ZanixCoreAsyncMQProvider`'s `enqueue`/`sendMessage`, the `@Subscriber` decorator, and the queue-handler validation flow. |
| `asyncmq-worker-and-tasks`          | `registerJob`'s `customQueue`-vs-`processingQueue` duality, the `runJob`/`runTask` dispatch split, and running the external worker process.         |
| `asyncmq-scheduling-and-cron`       | Delayed/future messages via `schedule`, and recurring execution via `registerCronJob`.                                                              |
| `asyncmq-dlq-reprocessing`          | `registerDLQProcessor` — the `@zanix/asyncmq/dlq` bridge that reprocesses `@zanix/datamaster`'s DLQ entries via cron.                               |

**`packages/space/`**

| Skill                           | Use it for                                                                                                                              |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `space-routing-and-rendering`   | File-based routing, layouts/loading/error segments, the document shell/contract, the not-found page, and `@zanix/space/testing`.        |
| `space-comets`                  | Selective hydration — `'use comet'`, `defineComet`, hydration timing, `persist`, `'server-only'` enforcement.                           |
| `space-orbit-navigation`        | Client-side navigation — `initOrbit`, prefetch config, `renderToResponse`/`useRequestCache`, the `initialState` serialization contract. |
| `space-middleware-and-security` | Zero-config CSP/security headers, `defineMiddleware`/`cspGuard`/`securityHeadersGuard`, and `csrfGuard`.                                |
| `space-i18n-and-population`     | `langPreHandler`/`langGuard`, `populationGuard`, and `loadMessages` — which content variant a request gets.                             |
| `space-head-and-seo`            | `<title>`/`<meta>`/`<link>` precedence/dedup, `buildCanonicalLink`/`buildHreflangLinks`, and `robots.txt`/`sitemap.xml`.                |
| `space-styling-and-theming`     | `cssPlugin`, the design-token convention, and `defineSpaceApp({ theme: { resolve } })`.                                                 |
| `space-assets-and-media`        | Static assets, content-hashed assets, image/SVG optimization, and video/audio transcoding.                                              |
| `space-pwa`                     | `pwaPlugin`/`defineSpaceApp({ pwa })`, icons, and the service worker strategy.                                                          |
| `space-validation`              | The build-time document validation system — severity/opt-in/strict, the `basis` field, and what's deliberately never checked.           |

**`packages/auth/`**

| Skill                                | Use it for                                                                                                                               |
| ------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `auth-oauth2`                        | `GoogleOAuth2Connector`, extending `OAuth2Connector` for a custom provider, and why the authorization-code flow beats the implicit flow. |
| `auth-jwt-and-sessions`              | Core registration, JWT create/verify/decode with key rotation, session token lifecycle, and refresh-token rotation.                      |
| `auth-otp-and-totp`                  | One-time-password delivery and authenticator-app 2FA.                                                                                    |
| `auth-permissions-and-rate-limiting` | `AuthTokenValidation` vs. `RequirePermissions`, the OR-not-AND semantics, and `rateLimitGuard`.                                          |
| `auth-network-security`              | `IpAllowlistGuard` — IP/CIDR restriction and the `trustProxyHeader` safety requirement.                                                  |
| `auth-service-credential`            | Machine-to-machine authentication — `createServiceAssertion`/`exchangeServiceCredential` and key rotation.                               |

**`packages/app/`**

| Skill                                | Use it for                                                                                                                                            |
| ------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `app-manifest-and-composition`       | `defineZanixApp()`, the manifest reference, `AppContainer`, `registerApp`/`ResourceRegistry`/`resolveResources`, and `activateApps`/`deactivateApps`. |
| `app-behaviors-and-overrides`        | `ctx.behavior()`/`resolveBehavior()`, style-only overrides, and the Configuration vs. Extension vs. Override decision table.                          |
| `app-remote-calls-and-control-plane` | The Control Plane, `ctx.remote()`, `allowedCallers`, Remote Resource Binding, and mTLS.                                                               |
| `app-leader-election-and-gateway`    | Automatic leader election for scheduled jobs, fencing tokens, Redlock, and the Gateway's routing.                                                     |
| `app-hot-install-and-multitenancy`   | `installApp`/`uninstallApp`, the route-unmount gotcha, jobs/events being restart-only, and multi-tenancy via naming.                                  |
| `app-mcp-composability`              | Exposing an operation as a Model Context Protocol tool.                                                                                               |
| `app-sandboxing`                     | Running an operation inside a dedicated, permission-restricted Deno Worker.                                                                           |
| `app-standalone-deployment`          | `bootstrapRemoteApp` — running one app as its own standalone remote process.                                                                          |
| `app-publishing`                     | Distributing a `defineZanixApp()` as a package for a different team's host.                                                                           |

**`packages/admin/`**

| Skill                             | Use it for                                                                                                                                         |
| --------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `admin-composition-and-extension` | `ZanixAdminHub.start()` vs. composing `defineAdminHubApp()`/`defineLocalAdminApp()` directly, and the Extension pattern's 3-step sub-app template. |
| `admin-service-registry`          | `ServiceRegistry` — the shared known-services list, and reachability validation.                                                                   |
| `admin-triggers-aggregator`       | `TriggersAggregator`'s fan-out/proxy, `TriggersController`, and `createTriggersAdminController`'s split.                                           |
| `admin-templates-api`             | The two-controller `/templates` composition and the `POST /templates/sync` extension.                                                              |
| `admin-service-authentication`    | The three-step service-to-service auth flow and `ZanixAdminHub.start({auth})`.                                                                     |

**`packages/core/`**

| Skill                      | Use it for                                                                                                                                                     |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `core-bootstrap-and-setup` | `Zanix.start()`/`bootstrap()`/`startWorker()`, named apps, and `Zanix.setup()`'s cross-cutting config (including the real `AssetService` `assets` constructs). |
| `core-admin-apis`          | `Zanix.start({admin})` — this service's own embedded admin server, its routes, ports, roles, and `ADMIN_SERVER_ID`.                                            |
| `core-admin-architecture`  | This service's admin API vs. the centralized `ZanixAdminHub` — two independent servers, and running both safely.                                               |

**`packages/utils/`**

| Skill                                     | Use it for                                                                                                                           |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `utils-validator-core`                    | `BaseRTO`, `classValidation`, `defineValidationDecorator`, and the `expose`/`optional`/`transform` mechanics every decorator shares. |
| `utils-validator-decorators`              | The full decorator catalog (strings/numbers/dates/arrays/enum/boolean) and their individual gotchas.                                 |
| `utils-errors`                            | `HttpError`/`ApplicationError`/`PermissionDenied`/`InternalError`, `serializeError`.                                                 |
| `utils-logger`                            | The default `Logger`, its six storage styles, and redaction defaults.                                                                |
| `utils-encryption-and-masking`            | AES/RSA/HMAC, salted hashing, and reversible/irreversible masking.                                                                   |
| `utils-workers`                           | `WorkerManager` — task registration, the `permissions` model, and its honest limitations.                                            |
| `utils-linter-plugins`                    | The four `Deno.lint.Plugin` exports and their rules.                                                                                 |
| `utils-config-and-project-helpers`        | Config file access, path helpers, the global `Znx` namespace, `mockWrap`, and what moved to `@zanix/cli`.                            |
| `utils-interpolation-and-data-transforms` | `interpolate`/`interpolateEnv` (the canonical `${{ENV_VAR}}`-missing footgun), route/URL transforms.                                 |
| `utils-concurrency-and-sync`              | `Semaphore`/`LockManager`, `nextCronDate`, and `planCodeSync`.                                                                       |
| `utils-encoding-and-network`              | Encoding helpers, `zanixConstants`/`zanixRegex`, and IP/CIDR utilities.                                                              |

`jsdoc-jsr-audit` and `docs-readme-audit` are scoped to publishable
**libraries** (anything with a `deno.json(c)` `"exports"` field) — they don't
apply to a deployed app/microservice that isn't published as a package. Most of
the rest apply to any project regardless of that distinction; each skill's own
description says which.

Each skill is self-contained in its own `SKILL.md`, distilled from real,
hands-on sessions — not written speculatively. They encode what actually needed
fixing, what turned out to be a false alarm, and the token-saving habits (batch
analysis, automated link/symbol/anchor checks instead of re-reading files,
`git stash` baselines) that made those sessions efficient.

## Agents

Under `.claude/agents/`, for work that's a full orchestrated flow (specific
tools, several steps, a verdict to report) rather than a question answered
inline mid-session. An agent always consumes skills for its actual rules — it
never re-encodes them in its own prompt.

| Agent                                          | Use it for                                                                                                                     |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `architecture-reviewer`                        | Reviewing a diff/PR in a Zanix **library** repo for cross-package dependency-direction or local-API-vs-aggregator violations before merging. |
| `consumer-conventions-reviewer`                | Reviewing a diff/PR in a **consumer** Zanix project for server-conventions/naming/envvar/observability violations — the consumer-side sibling of `architecture-reviewer`. |
| `zanix-feature-builder`                        | Adding a feature/artifact to an **existing consumer** Zanix project (`server`/`space`/`spacecraft`/`app`/`library`) — never a Zanix package itself. |
| `zanix-remote-api-app-builder`                 | Standing up a brand-new `zanix-remote-api-app-pattern`-shaped `@zanix/space` app from zero — scaffold, auth composition, and a first resource slice — then handing off to `zanix-feature-builder` for every later resource. |
| `zanix-issue-reporter`                         | Interactively filing a real GitHub issue via `zanix report-issue` on a human's behalf — no maintainer/consumer restriction.    |
| `packages/cli/cli-generator-expert`            | Adding or modifying a `zanix generate`/`zanix new` generator in `@zanix/cli`.                                                  |
| `packages/datamaster/datamaster-builder`       | Adding a connector slot, a model, or a trigger action type to `@zanix/datamaster`.                                             |
| `packages/notifications/notifications-builder` | Adding a base or derived template to `@zanix/notifications`.                                                                   |
| `packages/space-ui/space-ui-builder`           | Adding a new component to `@zanix/space-ui`, following the seven seams and existing build patterns.                            |
| `packages/utils/utils-builder`                 | Adding a new Deno.lint rule (or auto-fix to an existing one) to `@zanix/utils`'s zanix plugin, a new general-purpose helper, or a new validation decorator. |
| `packages/admin/admin-builder`                 | Adding a new sub-app to `@zanix/admin`'s hub/local composition, following the Extension pattern's 3-step template.             |
| `packages/auth/auth-builder`                   | Shipping a new OAuth2 provider connector for `@zanix/auth`, mirroring `GoogleOAuth2Connector`.                                 |
| `packages/server/server-builder`               | Adding/extending a cross-cutting mechanism inside `@zanix/server` itself — a new transport, core provider/connector slot, protocol, or middleware primitive category. |
| `dependency-drift`                             | Verifying a generated/documented claim against real, currently-published `@zanix/*` versions.                                  |
| `dependency-direction-sweep`                   | Periodic sweep across all repos for a real, existing dependency-direction/circular-import violation.                          |
| `dead-code-sweep`                              | Periodic sweep within one repo for orphaned modules, duplicate implementations, and stale references to either.                |
| `conventions-sweep`                            | Periodic sweep across all repos for a real, existing naming/envvar/observability convention violation.                        |
| `test-tier-sweep`                              | Periodic sweep across all repos for a test placed in the wrong `@tests/{unit,integration,functional}/` tier.                   |
| `benchmark-sweep`                              | Running a repo's own `bench`/`bench:baseline`/`test:perf`-shaped tasks and turning raw output into a pass/fail/informational report. |
| `release-manager`                              | Executing the CHANGELOG/version/branch/commit/push/tag sequence, including coordinated multi-repo releases.                    |
| `documentation-agent`                          | Auditing a package's own docs/JSDoc, and checking whether a _skill file_ has gone stale against real code.                     |
| `testing-validation`                           | Verifying real test coverage for a change, respecting a repo's own testing-tier convention when one exists.                    |
| `ecosystem-maintenance`                        | Periodic sweep across all repos for outdated/deprecated _third-party_ dependencies — a different axis from `dependency-drift`. |

For the full maintainer-vs-consumer audience breakdown of every agent and
skill listed here, see [CATALOG.md](./CATALOG.md).

Per-package extension agents (one per repo meeting the criterion in the
design artifact's §10, each orchestrating that repo's own
`packages/<repo>/` skills; named `<repo>-builder` unless a more specific
name — like `cli-generator-expert` — fits better) now cover `admin`,
`auth`, `cli`, `datamaster`, `notifications`, `server`, `space-ui`, and
`utils` (`utils-builder` narrowly covers lint rules/auto-fix, general
helpers, and validation decorators — not the error/logger/workers/
encryption subsystems, or `validator-core`'s own mechanism). `asyncmq`,
`space`, `app`, and `core` are confirmed, not just
"not yet built," to not need one — none has the repeatable multi-step
workflow the criterion requires; see the design artifact's §10 table for
the reasoning per repo.
`zanix-feature-builder` and `zanix-remote-api-app-builder` are the two
consumer-side counterparts to all of the above — neither ever touches a
Zanix library's own source. They split by lifecycle stage, not by which
package a consumer app talks to: `zanix-remote-api-app-builder` stands up a
brand-new `zanix-remote-api-app-pattern`-shaped `@zanix/space` app from
zero (scaffold, auth composition, a first resource slice proving it real),
then hands off; `zanix-feature-builder` owns every feature/resource added
to a consumer project from that point on, including every resource after
the first on an app the other agent stood up. Deliberately not called
"maintainer": that word is reserved for `ecosystem-maintenance`'s periodic
third-party dependency sweep, a different function entirely.

## Using these skills and agents

All skills are symlinked once into `~/.claude/skills` (user-level) — Claude Code
loads them there regardless of which project a session is launched from, so
nothing needs to be repeated or re-installed per repo:

```bash
ln -s /path/to/claude-skills/.claude/skills/universal/docs-readme-audit ~/.claude/skills/docs-readme-audit
```

Every skill under `universal/`, `ecosystem/`, and `packages/*/` is linked the
same way, one symlink per skill folder, all pointing back into this repo. That's
a deliberate trade-off: it means every skill is technically visible in every
session on this machine (including `packages/cli/*` while working in an
unrelated repo), rather than maintaining a separate, scoped `.claude/skills/`
install per project — each skill's own `description` is what Claude actually
uses to decide relevance, and duplicating the same symlink set across a dozen
repos was judged not worth the upkeep.

Once available, invoke a skill directly (e.g. `/docs-readme-audit`) or let
Claude pick it up automatically when a request matches its `description`.

Agents follow the identical pattern into `~/.claude/agents`, one symlink per
`.md` file:

```bash
ln -s /path/to/claude-skills/.claude/agents/architecture-reviewer.md ~/.claude/agents/architecture-reviewer.md
```

## Contributing

These skills are meant to keep evolving: when a session uncovers a new pattern,
gotcha, or token-saving trick worth generalizing, fold it into the relevant
`SKILL.md` (or add a new skill under the right tier — see
[Structure](#structure)) rather than letting it stay a one-off prompt in a
single project. When adding a new `packages/<repo>/` skill, keep it thin — it
should distill hard-won lessons, not duplicate that repo's own `docs/`
wholesale.

## License

MIT — see [LICENSE](./LICENSE).

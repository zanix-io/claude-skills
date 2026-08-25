# Catalog

A navigation aid over every agent and skill in this repo, tagged by who
actually reaches for it: someone extending a Zanix **library** package's own
source (`server`, `datamaster`, `notifications`, `auth`, `asyncmq`, `admin`,
`core`, `app`, `space`, `space-ui`, `cli`, `utils`), someone building an
application **on top of** the ecosystem without touching a library's
internals, or genuinely both. For what each skill/agent actually covers, see
[README.md](./README.md) — that's the one-line-per-item reference; this file
adds the audience dimension README doesn't track, plus the file/directory
layout so you can go straight to a `SKILL.md`/agent `.md` without grepping.

Four tags are used:

- **Maintainer** — extends a Zanix library package's own internals. The
  audience is whoever works inside that package's own repo.
- **Consumer** — used by someone building an app/service on the ecosystem,
  never touching a `~/Documents/Development/ZanixLibraries/<repo>` package's
  source.
- **Shared** — genuinely used by both, verified against a real dual-use
  statement (an agent's own scope section, or `zanix-feature-builder`'s
  "Skills to load" list explicitly routing a consumer task through it).
- **Review-Maintenance** — a periodic sweep or release/testing gate, not tied
  to adding a feature to one specific package; who it serves varies by which
  repo it's pointed at, noted per row.

The maintainer/consumer split itself isn't invented here — it's the same
split `zanix-feature-builder.md`'s own frontmatter draws against every
package `-builder` agent, and the one several agents state explicitly in
their own "Out of scope" language (`server-builder.md` naming
`cli-generator-expert` and "plain application code following
`zanix-server-conventions`" as look-alikes that aren't its job, `auth-builder`
excluding five of its own package's six topic skills by name). Where a
skill's real scope statement complicates the tier its directory would
suggest, that's called out below rather than smoothed over.

## Agents

`.claude/agents/` — flat files apply across the ecosystem; `packages/<repo>/`
agents extend one specific package.

One-line purpose for any of these is README's own job — see
[README.md](./README.md)'s Agents table; this one only adds audience +
which package a builder extends.

| Agent | Audience | Extends |
| --- | --- | --- |
| [`architecture-reviewer`](.claude/agents/architecture-reviewer.md) | Review-Maintenance (maintainer repos) | — |
| [`benchmark-sweep`](.claude/agents/benchmark-sweep.md) | Review-Maintenance (maintainer repos — today `server`/`space`) | — |
| [`consumer-conventions-reviewer`](.claude/agents/consumer-conventions-reviewer.md) | Review-Maintenance (consumer repos) | — |
| [`conventions-sweep`](.claude/agents/conventions-sweep.md) | Review-Maintenance (maintainer repos) | — |
| [`dead-code-sweep`](.claude/agents/dead-code-sweep.md) | Review-Maintenance (any repo — examples are all Zanix library repos, but nothing in scope restricts it to them) | — |
| [`dependency-direction-sweep`](.claude/agents/dependency-direction-sweep.md) | Review-Maintenance (maintainer repos) | — |
| [`dependency-drift`](.claude/agents/dependency-drift.md) | Review-Maintenance (Shared — checks both `@zanix/cli` output for consumer projects and skill claims) | — |
| [`documentation-agent`](.claude/agents/documentation-agent.md) | Review-Maintenance (package-docs half is maintainer-scoped to publishable packages; skill-staleness half applies to any skill, maintainer or consumer) | — |
| [`ecosystem-maintenance`](.claude/agents/ecosystem-maintenance.md) | Review-Maintenance (maintainer repos) | — |
| [`release-manager`](.claude/agents/release-manager.md) | Review-Maintenance (Shared — release mechanics apply the same way to a consumer's own publishable `library`/`app` project) | — |
| [`test-tier-sweep`](.claude/agents/test-tier-sweep.md) | Review-Maintenance (maintainer repos) | — |
| [`testing-validation`](.claude/agents/testing-validation.md) | Review-Maintenance (Shared) | — |
| [`zanix-feature-builder`](.claude/agents/zanix-feature-builder.md) | **Consumer** | — |
| [`zanix-remote-api-app-builder`](.claude/agents/zanix-remote-api-app-builder.md) | **Consumer** | — |
| [`zanix-issue-reporter`](.claude/agents/zanix-issue-reporter.md) | Shared | — |
| [`packages/admin/admin-builder`](.claude/agents/packages/admin/admin-builder.md) | **Maintainer** | admin |
| [`packages/auth/auth-builder`](.claude/agents/packages/auth/auth-builder.md) | **Maintainer** (narrowly — see the auth skills table below) | auth |
| [`packages/cli/cli-generator-expert`](.claude/agents/packages/cli/cli-generator-expert.md) | **Maintainer** | cli |
| [`packages/datamaster/datamaster-builder`](.claude/agents/packages/datamaster/datamaster-builder.md) | **Maintainer** | datamaster |
| [`packages/notifications/notifications-builder`](.claude/agents/packages/notifications/notifications-builder.md) | **Maintainer** | notifications |
| [`packages/server/server-builder`](.claude/agents/packages/server/server-builder.md) | **Maintainer** | server |
| [`packages/space-ui/space-ui-builder`](.claude/agents/packages/space-ui/space-ui-builder.md) | **Maintainer** | space-ui |
| [`packages/utils/utils-builder`](.claude/agents/packages/utils/utils-builder.md) | **Maintainer** (narrowly — lint rules/auto-fix, general helpers, and validation decorators only, not the error/logger/workers/encryption subsystems or `validator-core`'s own mechanism) | utils |

No maintainer builder exists yet for `asyncmq`, `space`, `app`, or `core` —
per README's own roadmap note, that's a deliberate gap (design artifact
§10's prioritized order), not an oversight in this catalog (`utils` DOES
have one, narrowly scoped — `utils-builder`, listed above). It
explains why several `packages/<repo>/` skill directories below don't split
as cleanly maintainer-vs-consumer as the ones with a builder do: nothing has
yet forced the maintainer-only sections of e.g. the `space`/`asyncmq` skills
to be pulled apart from their consumer-usage sections the way `auth-builder`'s
explicit exclusion list did for `auth`.

## Skills

Grouped by physical tier, matching the repo's own `.claude/skills/`
structure. One-line purpose is each skill's own frontmatter `description`
(condensed); see [README.md](./README.md) for the fuller version and any
skill's own `SKILL.md` for the real thing.

### `universal/` — 7 skills, all Shared

Stack-agnostic engineering discipline; nothing here is Zanix-specific, so
the maintainer/consumer split doesn't apply to any of them individually.

| Skill | Audience |
| --- | --- |
| [`complete-test-coverage`](.claude/skills/universal/complete-test-coverage/SKILL.md) | Shared |
| [`docs-readme-audit`](.claude/skills/universal/docs-readme-audit/SKILL.md) | Shared (scoped to publishable packages — see README's own caveat paragraph) |
| [`documentation-voice`](.claude/skills/universal/documentation-voice/SKILL.md) | Shared |
| [`feature-completeness-conventions`](.claude/skills/universal/feature-completeness-conventions/SKILL.md) | Shared |
| [`jsdoc-jsr-audit`](.claude/skills/universal/jsdoc-jsr-audit/SKILL.md) | Shared (scoped to publishable packages) |
| [`naming-and-structure-conventions`](.claude/skills/universal/naming-and-structure-conventions/SKILL.md) | Shared — its own description flags that its "Known current gaps" list is specific to the 12 audited library repos and doesn't describe a consumer project; the casing rules themselves apply identically to both |
| [`release-management`](.claude/skills/universal/release-management/SKILL.md) | Shared |

### `ecosystem/` — 11 skills

Rules that cross more than one Zanix package. This is the tier the task's
own ground truth targets, and it's genuinely mixed, not uniformly one tag.

| Skill | Audience | Note |
| --- | --- | --- |
| [`skill-and-agent-authoring`](.claude/skills/ecosystem/skill-and-agent-authoring/SKILL.md) | **Maintainer** | About this repo's own process — tier placement, symlinking, validation. Not meaningful outside working on `claude-skills` itself. |
| [`zanix-agent-dispatch-discipline`](.claude/skills/ecosystem/zanix-agent-dispatch-discipline/SKILL.md) | Shared | The dispatch-safety mechanics (named-agent gate, new-type registration delay, concurrent-dispatch verification, cross-session coordination) apply the same way regardless of whether the agent being dispatched is maintainer- or consumer-facing. |
| [`zanix-dependency-direction`](.claude/skills/ecosystem/zanix-dependency-direction/SKILL.md) | Shared | Own description explicitly includes `@zanix/app`/`@zanix/space` "as consumers of the same tiers"; `zanix-feature-builder` loads it for any new cross-package import in a consumer project. |
| [`zanix-envvar-conventions`](.claude/skills/ecosystem/zanix-envvar-conventions/SKILL.md) | Shared | `zanix-feature-builder` loads it verbatim, "same discipline the maintainer '-builder' agents apply, not a lighter version because this is a consumer project." |
| [`zanix-issue-reporting`](.claude/skills/ecosystem/zanix-issue-reporting/SKILL.md) | Shared | Filing via `zanix report-issue` needs no repo write access, so it applies the same way to a maintainer agent and a consumer-side one — loaded by all 19 other agents plus the interactive `zanix-issue-reporter`. |
| [`zanix-local-api-implementation`](.claude/skills/ecosystem/zanix-local-api-implementation/SKILL.md) | **Maintainer** | Mechanics of a new `<domain>-api/` subpath — only meaningful inside a Zanix library package. |
| [`zanix-local-api-vs-aggregator`](.claude/skills/ecosystem/zanix-local-api-vs-aggregator/SKILL.md) | **Maintainer** | "Use before writing a new controller in any Zanix library package" — its own description. |
| [`zanix-observability-conventions`](.claude/skills/ecosystem/zanix-observability-conventions/SKILL.md) | Shared | Own description: "in a library repo or a consumer project," explicitly. |
| [`zanix-remote-api-app-pattern`](.claude/skills/ecosystem/zanix-remote-api-app-pattern/SKILL.md) | **Consumer** | "Use when scaffolding or reviewing a `@zanix/space` app that consumes another service's admin/API surface" — its own description; grounded in `@zanix/console` as the cited precedent, not the pattern's own subject. |
| [`zanix-server-conventions`](.claude/skills/ecosystem/zanix-server-conventions/SKILL.md) | **Consumer** | "Building/reviewing backend microservices **on** `@zanix/server`" — the consumer-side counterpart `server-builder` itself points to for anything that isn't extending `@zanix/server`'s own internals. |
| [`zanix-test-tier-conventions`](.claude/skills/ecosystem/zanix-test-tier-conventions/SKILL.md) | Shared | Every real Zanix repo (library and consumer) has a real `@tests/{unit,integration,functional}/` structure — loaded by every builder plus `testing-validation`/`zanix-feature-builder`, refined by a repo's own more specific testing skill (`cli-generator-testing`) when one exists. |

### `packages/<repo>/` — 73 skills

Most of this tier documents one package's own internals, but "internals" for
`app`/`core` means the runtime surface a consumer boots into, not something
extended from outside — so those two directories flip the usual expectation.
Read each group's note before assuming directory location alone decides the
tag.

**`packages/admin/`** (5 skills) — mostly Maintainer; this is
`@zanix/admin`'s own composition, not something a generic consumer app calls
into except at the one auth seam.

| Skill | Audience |
| --- | --- |
| [`admin-composition-and-extension`](.claude/skills/packages/admin/admin-composition-and-extension/SKILL.md) | Maintainer |
| [`admin-service-authentication`](.claude/skills/packages/admin/admin-service-authentication/SKILL.md) | Shared — any service (including a consumer's own) authenticating against `/admin/*` or `ZanixAdminHub` itself uses the "reverse direction" half |
| [`admin-service-registry`](.claude/skills/packages/admin/admin-service-registry/SKILL.md) | Maintainer |
| [`admin-templates-api`](.claude/skills/packages/admin/admin-templates-api/SKILL.md) | Maintainer |
| [`admin-triggers-aggregator`](.claude/skills/packages/admin/admin-triggers-aggregator/SKILL.md) | Maintainer |

**`packages/app/`** (9 skills) — **Consumer**, all of them. Unlike
`datamaster`/`auth`/`space-ui`, nothing here is framed as extending
`@zanix/app`'s own internals — every description is about authoring/
composing/deploying a Zanix App on top of the package's public API
(`defineZanixApp()`, `ctx.behavior()`, `ctx.remote()`, `installApp`, etc.),
and there's no `app-builder` agent. Flagged explicitly below as the clearest
case where `packages/<repo>/` location doesn't predict "maintainer."

| Skill |
| --- |
| [`app-behaviors-and-overrides`](.claude/skills/packages/app/app-behaviors-and-overrides/SKILL.md) |
| [`app-hot-install-and-multitenancy`](.claude/skills/packages/app/app-hot-install-and-multitenancy/SKILL.md) |
| [`app-leader-election-and-gateway`](.claude/skills/packages/app/app-leader-election-and-gateway/SKILL.md) |
| [`app-manifest-and-composition`](.claude/skills/packages/app/app-manifest-and-composition/SKILL.md) |
| [`app-mcp-composability`](.claude/skills/packages/app/app-mcp-composability/SKILL.md) |
| [`app-publishing`](.claude/skills/packages/app/app-publishing/SKILL.md) |
| [`app-remote-calls-and-control-plane`](.claude/skills/packages/app/app-remote-calls-and-control-plane/SKILL.md) |
| [`app-sandboxing`](.claude/skills/packages/app/app-sandboxing/SKILL.md) |
| [`app-standalone-deployment`](.claude/skills/packages/app/app-standalone-deployment/SKILL.md) |

**`packages/asyncmq/`** (4 skills) — Consumer-leaning; no `asyncmq-builder`
agent exists (see the memory note recorded during this ecosystem's work:
`@zanix/asyncmq` is hardwired to AMQP with only one provider slot, so there
isn't a repeatable "add a provider" maintainer workflow the way `datamaster`
has). All four descriptions read as usage of an already-built mechanism
(`ZanixRabbitMQConnector`, `registerJob`, `schedule`, `registerDLQProcessor`).
`zanix-feature-builder` still routes consumer `asyncmq` work through them
with the same "consumer lens" caveat it applies to `datamaster`/`space-ui`,
which is the one signal keeping these tagged Shared rather than pure
Consumer.

| Skill | Audience |
| --- | --- |
| [`asyncmq-connector-and-subscribers`](.claude/skills/packages/asyncmq/asyncmq-connector-and-subscribers/SKILL.md) | Shared (consumer-leaning) |
| [`asyncmq-dlq-reprocessing`](.claude/skills/packages/asyncmq/asyncmq-dlq-reprocessing/SKILL.md) | Shared (consumer-leaning) |
| [`asyncmq-scheduling-and-cron`](.claude/skills/packages/asyncmq/asyncmq-scheduling-and-cron/SKILL.md) | Shared (consumer-leaning) |
| [`asyncmq-worker-and-tasks`](.claude/skills/packages/asyncmq/asyncmq-worker-and-tasks/SKILL.md) | Shared (consumer-leaning) |

**`packages/auth/`** (6 skills) — genuinely split, not uniform. Only one of
the six is inside `auth-builder`'s stated maintainer scope; the other five
are explicitly named as *out* of that scope, in `auth-builder`'s own
frontmatter, as "review/config work against fixed mechanics."

| Skill | Audience |
| --- | --- |
| [`auth-oauth2`](.claude/skills/packages/auth/auth-oauth2/SKILL.md) | Shared — the one `auth-builder` maintainer workflow (new provider connector); also usable by a consumer app writing its own `OAuth2Connector` subclass without touching `@zanix/auth`'s source |
| [`auth-jwt-and-sessions`](.claude/skills/packages/auth/auth-jwt-and-sessions/SKILL.md) | Consumer — named out of `auth-builder`'s scope |
| [`auth-network-security`](.claude/skills/packages/auth/auth-network-security/SKILL.md) | Consumer — named out of `auth-builder`'s scope |
| [`auth-otp-and-totp`](.claude/skills/packages/auth/auth-otp-and-totp/SKILL.md) | Consumer — named out of `auth-builder`'s scope |
| [`auth-permissions-and-rate-limiting`](.claude/skills/packages/auth/auth-permissions-and-rate-limiting/SKILL.md) | Consumer — named out of `auth-builder`'s scope |
| [`auth-service-credential`](.claude/skills/packages/auth/auth-service-credential/SKILL.md) | Consumer — named out of `auth-builder`'s scope |

**`packages/cli/`** (5 skills) — all **Maintainer**; `cli-generator-expert`'s
entire job is these five topics (command wiring, generators, scaffold
assembly, dependency compatibility, generator testing).

| Skill |
| --- |
| [`cli-artifact-generators`](.claude/skills/packages/cli/cli-artifact-generators/SKILL.md) |
| [`cli-command-architecture`](.claude/skills/packages/cli/cli-command-architecture/SKILL.md) |
| [`cli-dependency-compatibility`](.claude/skills/packages/cli/cli-dependency-compatibility/SKILL.md) |
| [`cli-generator-testing`](.claude/skills/packages/cli/cli-generator-testing/SKILL.md) |
| [`cli-scaffold-assembly`](.claude/skills/packages/cli/cli-scaffold-assembly/SKILL.md) |

**`packages/core/`** (3 skills) — **Consumer**, all of them, same reasoning
as `packages/app/`: `Zanix.start()`/`Zanix.setup()`/`admin` are the public
surface any Zanix service boots through, not `@zanix/core`'s own internals.
No `core-builder` agent exists.

| Skill |
| --- |
| [`core-admin-apis`](.claude/skills/packages/core/core-admin-apis/SKILL.md) |
| [`core-admin-architecture`](.claude/skills/packages/core/core-admin-architecture/SKILL.md) |
| [`core-bootstrap-and-setup`](.claude/skills/packages/core/core-bootstrap-and-setup/SKILL.md) |

**`packages/datamaster/`** (8 skills) — all Shared. `datamaster-builder`'s
own frontmatter explicitly claims both "genuinely repeatable extension
workflows" (connector slots, trigger action types) **and** "routine work
across data protection, DLQ, storage, cache, and observability" — i.e., all
eight topics, not a subset the way `auth-builder` excludes five of six.
`datamaster-connector-registration` still leans more maintainer in practice
(it's specifically about adding a *new* slot); `zanix-feature-builder`
applies it to a consumer's own `library`-type project only as a general
pattern, not `@zanix/datamaster`-specific usage.

| Skill | Audience |
| --- | --- |
| [`datamaster-cache`](.claude/skills/packages/datamaster/datamaster-cache/SKILL.md) | Shared |
| [`datamaster-connector-registration`](.claude/skills/packages/datamaster/datamaster-connector-registration/SKILL.md) | Shared (maintainer-leaning) |
| [`datamaster-data-protection`](.claude/skills/packages/datamaster/datamaster-data-protection/SKILL.md) | Shared |
| [`datamaster-database-and-models`](.claude/skills/packages/datamaster/datamaster-database-and-models/SKILL.md) | Shared |
| [`datamaster-dlq`](.claude/skills/packages/datamaster/datamaster-dlq/SKILL.md) | Shared |
| [`datamaster-observability`](.claude/skills/packages/datamaster/datamaster-observability/SKILL.md) | Shared |
| [`datamaster-storage`](.claude/skills/packages/datamaster/datamaster-storage/SKILL.md) | Shared |
| [`datamaster-triggers`](.claude/skills/packages/datamaster/datamaster-triggers/SKILL.md) | Shared |

**`packages/notifications/`** (5 skills) — mostly Shared;
`notifications-builder`'s scope names templates and connector/provider work
directly. `notifications-template-storage-modes` (choosing Mode A/B/C) reads
more like a deployment/config decision than package extension, so it's
tagged consumer-leaning.

| Skill | Audience |
| --- | --- |
| [`notifications-connectors`](.claude/skills/packages/notifications/notifications-connectors/SKILL.md) | Shared |
| [`notifications-provider`](.claude/skills/packages/notifications/notifications-provider/SKILL.md) | Shared |
| [`notifications-template-inheritance`](.claude/skills/packages/notifications/notifications-template-inheritance/SKILL.md) | Shared |
| [`notifications-template-storage-modes`](.claude/skills/packages/notifications/notifications-template-storage-modes/SKILL.md) | Shared (consumer-leaning) |
| [`notifications-templates`](.claude/skills/packages/notifications/notifications-templates/SKILL.md) | Shared |

**`packages/server/`** (1 skill) — **Maintainer**, explicitly. Its own
opening line contrasts itself with `zanix-server-conventions`: "Not for
application code."

| Skill |
| --- |
| [`zanix-server-internals`](.claude/skills/packages/server/zanix-server-internals/SKILL.md) |

**`packages/space/`** (10 skills) — **Consumer**-leaning, tagged as such
here even though `zanix-feature-builder` groups it with the same
"consumer-lens" caveat it gives `space-ui`/`datamaster`. Every one of the ten
descriptions reads as page/route/asset/CSS/i18n *usage* inside a space app;
none is framed as extending `@zanix/space`'s own rendering engine, and — like
`asyncmq` — no `space-builder` agent exists yet (README's own roadmap flags
`@zanix/space` as the one repo with an open benchmark-validation question
still blocking its builder agent). Once that agent exists, expect this group
to split the way `auth`'s six skills did.

| Skill |
| --- |
| [`space-assets-and-media`](.claude/skills/packages/space/space-assets-and-media/SKILL.md) |
| [`space-comets`](.claude/skills/packages/space/space-comets/SKILL.md) |
| [`space-head-and-seo`](.claude/skills/packages/space/space-head-and-seo/SKILL.md) |
| [`space-i18n-and-population`](.claude/skills/packages/space/space-i18n-and-population/SKILL.md) |
| [`space-middleware-and-security`](.claude/skills/packages/space/space-middleware-and-security/SKILL.md) |
| [`space-orbit-navigation`](.claude/skills/packages/space/space-orbit-navigation/SKILL.md) |
| [`space-pwa`](.claude/skills/packages/space/space-pwa/SKILL.md) |
| [`space-routing-and-rendering`](.claude/skills/packages/space/space-routing-and-rendering/SKILL.md) |
| [`space-styling-and-theming`](.claude/skills/packages/space/space-styling-and-theming/SKILL.md) |
| [`space-validation`](.claude/skills/packages/space/space-validation/SKILL.md) |

**`packages/space-ui/`** (6 skills) — split, verified against each skill's
real (not README-condensed) description, not just its directory.

| Skill | Audience |
| --- | --- |
| [`space-ui-architecture`](.claude/skills/packages/space-ui/space-ui-architecture/SKILL.md) | Maintainer — "Use before designing any **new** stateful component"; this is `space-ui-builder`'s gate, not usage |
| [`space-ui-component-patterns`](.claude/skills/packages/space-ui/space-ui-component-patterns/SKILL.md) | Maintainer — its real description (not README's shorter one) opens with "The discipline for building a **new** `@zanix/space-ui` component" |
| [`space-ui-foundation-primitives`](.claude/skills/packages/space-ui/space-ui-foundation-primitives/SKILL.md) | Shared (maintainer-leaning) — the extraction-criterion half is maintainer work; "what's public" is consumer-usable |
| [`space-ui-icons`](.claude/skills/packages/space-ui/space-ui-icons/SKILL.md) | Shared (consumer-leaning) — `createCatalogIcon` for "building your own [catalog]" is explicit consumer usage |
| [`space-ui-richtext`](.claude/skills/packages/space-ui/space-ui-richtext/SKILL.md) | Shared |
| [`space-ui-styling`](.claude/skills/packages/space-ui/space-ui-styling/SKILL.md) | Shared |

**`packages/utils/`** (11 skills) — all Shared. `utils-validator-core`'s own
description calls `BaseRTO` "the most-consumed module in the ecosystem,
since every Zanix package handling incoming data builds on it" — every
skill in this group documents primitives imported identically by library
packages and consumer apps alike.

| Skill |
| --- |
| [`utils-concurrency-and-sync`](.claude/skills/packages/utils/utils-concurrency-and-sync/SKILL.md) |
| [`utils-config-and-project-helpers`](.claude/skills/packages/utils/utils-config-and-project-helpers/SKILL.md) |
| [`utils-encoding-and-network`](.claude/skills/packages/utils/utils-encoding-and-network/SKILL.md) |
| [`utils-encryption-and-masking`](.claude/skills/packages/utils/utils-encryption-and-masking/SKILL.md) |
| [`utils-errors`](.claude/skills/packages/utils/utils-errors/SKILL.md) |
| [`utils-interpolation-and-data-transforms`](.claude/skills/packages/utils/utils-interpolation-and-data-transforms/SKILL.md) |
| [`utils-linter-plugins`](.claude/skills/packages/utils/utils-linter-plugins/SKILL.md) |
| [`utils-logger`](.claude/skills/packages/utils/utils-logger/SKILL.md) |
| [`utils-validator-core`](.claude/skills/packages/utils/utils-validator-core/SKILL.md) |
| [`utils-validator-decorators`](.claude/skills/packages/utils/utils-validator-decorators/SKILL.md) |
| [`utils-workers`](.claude/skills/packages/utils/utils-workers/SKILL.md) |

`utils-linter-plugins` is the one exception to "all Shared" above, since it
now also covers authoring: configuring/reviewing the four plugins is
Shared (any project's own `deno.json(c)`), but its "Authoring a new rule"/
"Adding auto-fix" sections — orchestrated by `packages/utils/utils-builder`
— are Maintainer-only, since only someone extending `@zanix/utils` itself
would add a rule to its own plugin.

## Flagged findings

- **`packages/app/` and `packages/core/` invert the usual `packages/<repo>/`
  expectation.** Every other package directory documents either extending
  the package (when a builder agent exists) or a genuine mix; these two
  document a consumer's usage of the package's own public runtime API
  (`defineZanixApp()`, `Zanix.start()`) with nothing framed as extending
  `@zanix/app`/`@zanix/core` internals, and neither has a builder agent.
  Worth keeping in mind before assuming "under `packages/`" means
  "maintainer-tier" anywhere else in this repo.
- **`auth-builder`'s frontmatter is the most precise scope statement in the
  repo** — it names five of its own package's six topic skills as explicitly
  out of its maintainer scope ("review/config work against fixed
  mechanics"). No other package builder draws that line as sharply;
  `datamaster-builder` claims all eight of its package's topics, and
  `space-ui-builder`'s "seven seams"/build-discipline framing effectively
  claims its two architecture-adjacent skills too. Treat `auth-builder`'s
  exclusion list as the model to check against if any future package skill
  set looks like it might also be over-claimed as "maintainer."
- **`space` and `asyncmq` currently read as consumer-leaning despite
  `zanix-feature-builder` filing them under the same "consumer-lens" caveat
  as `datamaster`/`space-ui`/`auth`/`notifications`.** That caveat implies
  latent maintainer content worth filtering out, but neither package has a
  builder agent to have forced that split into the open the way
  `auth-builder` did for `auth`. Not a contradiction to fix now — just a
  case where "depends on the task" is the honest answer, flagged rather than
  forced into a clean tag.
- **README.md's own "What's here" tables list every live, symlinked skill**,
  including `naming-and-structure-conventions` (`universal/`),
  `zanix-envvar-conventions`, and `zanix-observability-conventions` (both
  `ecosystem/`) — all three exist on disk, are linked into
  `~/.claude/skills/`, and are referenced by name from
  `zanix-feature-builder.md`'s own "Skills to load" section, consistent with
  their table rows. README's distribution-model description (the
  `~/.claude/skills`/`~/.claude/agents` symlink structure, one symlink per
  skill/agent folder) matches the real symlinks on this machine — no drift
  found there.

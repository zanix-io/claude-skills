---
name: zanix-remote-api-app-e2e-validation
description: Runbook for standing up a REAL, unmocked instance of the zanix-remote-api-app-pattern — a business service, a zanix-admin hub, and a @zanix/space consumer app (@zanix/console is the concrete grounding) — as three real processes talking real HTTP, then driving it with curl through login, the services list, and a full Triggers/Templates CRUD cycle. Applies equally to a maintainer regression-proofing a library-level change (ZanixAdminHub's serviceToken composition, admin-hub-auth's credential exchange) and to a consumer team standing up their OWN hub/service/app trio before shipping — the hub is something any team following the pattern runs themselves, not a single Zanix-owned service. Every existing test in this area mocks the hub client factories, so this is the only path that exercises the real network boundary. Not for day-to-day feature work on console itself (zanix-feature-builder's job) or for the pattern's own architecture (zanix-remote-api-app-pattern's job).
---

Grounded in a real run (2026-08-24): three live processes (a Mongo-backed
"business service" via `@zanix/core`'s `admin: true` shortcut, a real
`ZanixAdminHub`, and a real `@zanix/console` instance) wired together and
driven end to end with `curl` — not `deno test`, not a mocked hub-client
factory. That run is what surfaced the `serviceToken` composition gap
`@zanix/admin` later closed (see `zanix-remote-api-app-pattern` and
`@zanix/admin`'s own `admin-hub-app.ts`/`start.ts` docs) and two smaller,
still-open findings noted below. Every existing automated test in this area
— `console`'s own `src/@tests/unit/clients/`, `@zanix/admin`'s own hub
tests — mocks the network boundary this runbook exercises for real; that's
exactly the gap it exists to cover, and exactly why it can't be replaced by
"just run the test suite."

## Golden rule (token savings)

- **This is a runbook, not a feature-development guide.** Its only job is
  proving a real instance of the pattern still works end to end — reach for
  `zanix-remote-api-app-pattern` for the architecture, `zanix-feature-builder`
  for adding a resource to an existing app.
- **Use `zanix credentials mesh`, never hand-rolled `generateRSAKeys()`
  scripting**, for every identity's keypair below. The original 2026-08-24
  run predates that command and improvised a one-off Python script instead
  — don't replicate that; the command exists specifically to remove that
  tedium.
- **Compose the hub with `ZanixAdminHub.start({ serviceToken: true })`,
  never the manual `defineZanixApp`/`ProgramModule.defineApplication`
  workaround** shown as a historical artifact below. The original run
  predates `serviceToken` and had to hand-compose
  `createServiceExchangeController()` onto `ADMIN_HUB_APPLICATION` itself —
  that workaround is now dead weight; `serviceToken: true` is the same
  outcome in one option.
- **Re-verify every version pin and env-var name against the real current
  code before running this** — the exact `@zanix/*` versions and file
  paths below are what resolved on 2026-08-24; a stale pin here fails loud
  (a real `deno check`/import error), never silently.

## The three-process topology

| Process | What it is | Real composition |
| --- | --- | --- |
| Business service | A minimal Mongo-backed service exposing its own local admin API — the thing the hub aggregates. | `Zanix.start({ admin: { rest: { port: <p> } } })` — `@zanix/core`'s `admin: true` shortcut, same shape any real Zanix service gets. |
| Admin hub | The `zanix-admin` aggregator/proxy the console talks to. | `ZanixAdminHub.start({ rest: { port: <p> }, serviceToken: true, validateRegistry: true, auth: { serviceId: '<hub-id>' } })` — one call, one base URL for `/triggers`, `/templates`, `/registry/list`, AND `/admin/service-token`. |
| Console | The `@zanix/space` consumer app under validation. | `zanix space dev` / the project's own `start` task, pointed at the hub's base URL. |

The business service needs its own registered model for the aggregator to
have something real to fan out to — the 2026-08-24 run used:

```ts
import { registerModel } from '@zanix/datamaster'
registerModel({ name: 'demo-items', definition: { title: String, active: Boolean } })
```

**Confirm the business service's real local-admin REST prefix empirically
before wiring the hub's registry entry — don't assume a pattern.** The
2026-08-24 run's first guess at the business service's `adminBaseUrl` (a
bare `http://localhost:<port>`) was wrong; the real reachable path carried
an extra prefix segment the anchoring convention produced, only found by
actually curling the business service directly and adjusting
`ZANIX_ADMIN_SERVICES`'s `adminBaseUrl` afterward. Budget for this as a
real step, not a formality.

## Setting up real credentials

Three cooperating identities need real, cross-referenced RSA keypairs: the
business service, the hub, and console. Generate all three in one shot —
this command's hard minimum is 2 identities by design (a single service has
nothing to exchange a credential with):

```bash
cd ~/Documents/Development/ZanixLibraries/cli
deno task znx credentials mesh business-service admin-hub console
# or: deno run -A mod.ts credentials mesh business-service admin-hub console
```

This prints ready-to-paste `JWK_PRI_<id>`/`JWK_PUB_<id>`/
`SERVICE_PERMISSIONS_<id>` blocks for each identity, real keys, nothing
written to disk (see the command's own doc — local-dev/first-integration
convenience only, never a production secrets path). Paste each identity's
own block into its own `.env`, plus its counterparts' `JWK_PUB_<id>`/
`SERVICE_PERMISSIONS_<id>` entries for whichever peers it needs to
authenticate. The real, confirmed shape each `.env` needs (var names
current as of `console/src/auth/constants.ts`,
`console/src/clients/constants.ts`, `@zanix/admin`'s own registry/auth
options):

**Business service `.env`**
```
MONGO_URI=mongodb://localhost:27017/zanix_e2e_business
JWT_KEY=business-service-hmac-secret
JWK_PRI=<business service's own private key>
JWK_PUB=<business service's own public key>
JWK_PUB_zanix-admin-hub=<hub's public key>
SERVICE_PERMISSIONS_zanix-admin-hub=zanix:admin
ADMIN_SERVER_ID=biz
```

**Admin hub `.env`** — no `ADMIN_SERVER_ID`/second local-admin app needed
now that `serviceToken: true` composes `/admin/service-token` directly;
this is the one real simplification the old workaround's `.env` doesn't
show.
```
MONGO_URI=mongodb://localhost:27017/zanix_e2e_hub
JWT_KEY=admin-hub-hmac-secret
JWK_PRI=<hub's own private key>
JWK_PUB=<hub's own public key>
JWK_PUB_zanix-console=<console's public key>
SERVICE_PERMISSIONS_zanix-console=zanix:admin
JWK_PRI_zanix-admin-hub=<hub's own private key, keyed by its own service id>
ADMIN_HUB_SERVER_ID=hub
ZANIX_ADMIN_SERVICES=[{"serviceId":"business-service","adminBaseUrl":"http://localhost:9200/<real confirmed prefix>"}]
```

**Console `.env`**
```
JWT_KEY=console-hmac-secret
CONSOLE_OPERATOR_EMAIL=operator@example.com
CONSOLE_OPERATOR_PASSWORD_HASH='<output of generateHash(), single-quoted>'
JWK_PRI_zanix-console=<console's own private key>
ADMIN_HUB_BASE_URL=http://localhost:9100
```

**A real Deno `--env-file` footgun**: `CONSOLE_OPERATOR_PASSWORD_HASH`
(from `generateHash()`) always contains a literal `$` — its format is
`<hexSalt>$<base64Hash>`, never optional — and Deno's `--env-file` does
dotenv-style `$VAR` expansion, silently truncating the hash at that `$`
and breaking login with no clear error. Single-quote the value in `.env`,
as shown above, or it silently breaks every time, not just sometimes.

## Launching the three processes

Each process is a real `deno run` with the full permission set its own
`start`/`dev` task already declares — don't shortcut this to `--allow-all`
just for a throwaway run, it hides which permission actually matters.
Background each with `nohup ... &`, one log file per process:

```bash
# Business service (port 9200)
nohup deno run --min-dep-age=0 --env-file=business-service/.env \
  --allow-net --allow-env --allow-read --allow-sys --allow-write --allow-ffi \
  --no-prompt business-service/main.ts > business-service.log 2>&1 &

# Admin hub (port 9100) — real composition, current API:
#   const hubServers = await ZanixAdminHub.start({
#     rest: { port: 9100 },
#     serviceToken: true,
#     validateRegistry: true,
#     auth: { serviceId: 'zanix-admin-hub' },
#   })
nohup deno run --min-dep-age=0 --env-file=admin-hub/.env \
  --allow-net --allow-env --allow-read --allow-sys --allow-write --allow-ffi \
  --no-prompt admin-hub/main.ts > admin-hub.log 2>&1 &

# Console — its own real `deno.json` "start" task, e.g.:
nohup deno run --min-dep-age=0 --env-file=.env \
  --allow-net --allow-env --allow-read --allow-sys --allow-write --allow-ffi \
  --allow-run=ffmpeg,ffprobe --no-prompt mod.ts > console.log 2>&1 &
```

Tail each log for its own ready line before driving anything — `deno
run`'s own JSR/npm resolution on a cold cache takes real time, and hitting
console before the hub is listening produces a `RestClientError`/502 that
looks like a real bug but is just a startup race.

**Historical artifact — do not reproduce.** Before `serviceToken: true`
existed, the hub side needed a hand-composed `defineZanixApp` registering
`createServiceExchangeController()` directly under `ADMIN_HUB_APPLICATION`
via `ProgramModule.defineApplication`, wired manually alongside
`defineAdminHubApp`/`getAdminHubSubApps`. `ZanixAdminHub.start({ ...,
serviceToken: true })` is that exact outcome in one option now — never
hand-compose this again.

## Driving it for real — the curl walkthrough

Console's login is a real CSRF-protected form flow, not a bare POST — GET
first for the cookie + token, then POST with both:

```bash
BASE=http://localhost:20202   # console's own port
rm -f cookies.txt
curl -s -c cookies.txt -D headers.txt -o body.html "$BASE/login"
CSRF=$(grep -o 'name="_csrf" value="[^"]*"' body.html | sed -E 's/.*value="([^"]*)"/\1/')

curl -s -b cookies.txt -c cookies.txt -D login-headers.txt -o /dev/null "$BASE/login" \
  -X POST --data-urlencode "email=operator@example.com" \
  --data-urlencode "password=<the real password behind CONSOLE_OPERATOR_PASSWORD_HASH>" \
  --data-urlencode "_csrf=$CSRF" -H "X-Znx-Cookies-Accepted: true" \
  --max-redirs 0 -w "STATUS:%{http_code}\n"
```

A 302 with a fresh `set-cookie` header is a real login. From there, drive
the real resource flows with the same `-b cookies.txt` session:

- `GET /services` — confirm the business service shows up (proves the
  hub's registry + reachability check actually work, not just that the hub
  process is up).
- **Triggers, full CRUD**: `GET /triggers/new` (grab a fresh `_csrf`) →
  `POST /triggers/new` (`serviceId`, `model`, `active`, `triggers`) → `GET
  /triggers/<serviceId>/<model>` → `POST .../edit` → `POST` the detail page
  again with no body to delete → re-`GET` the detail page and confirm it's
  a clean not-found, not a stack trace.
- **Templates, full CRUD** — same shape, one level of nesting different:
  `GET /templates/new` → `POST` with `channel`/`name`/`hbs`/`description` →
  `GET /templates/<channel>/<name>` → `POST .../edit` → delete → re-`GET`.
- **Negative cases worth checking every run**: no session → 401/redirect
  on every protected route; a bad `/admin/service-token` body → a clean
  4xx, not a 500; a nonexistent `<serviceId>/<model>` detail URL → still
  worth a fresh look each run (see the known gap below).

## Known gaps this runbook will keep surfacing — not this skill's job to fix

- **No page-level error handling for a failed hub call.** A
  trigger/template detail URL for a resource that no longer exists returns
  a raw `RestClientError`/502 JSON blob straight to the browser today,
  instead of a real not-found/error page. Confirmed real, not yet fixed —
  an app-wide error-boundary design decision for console's `loader`/
  `action` layer, not a one-line patch. Don't rediscover this from scratch
  each run; check whether it's still open before filing it again.
- **The `--env-file` `$`-expansion footgun** documented above will keep
  resurfacing for anyone who forgets to single-quote a generated password
  hash — worth carrying into console's own `.env.example`/setup docs
  directly, not just here.

## Checklist

- [ ] All three identities' keys came from `zanix credentials mesh`, not a
      hand-rolled script.
- [ ] The hub is composed via `ZanixAdminHub.start({ serviceToken: true })`
      (or `defineAdminHubApp({ serviceToken: true })` directly) — no manual
      `createServiceExchangeController()` composition.
- [ ] The business service's real local-admin REST prefix was confirmed by
      curling it directly, not assumed, before wiring `ZANIX_ADMIN_SERVICES`.
- [ ] `CONSOLE_OPERATOR_PASSWORD_HASH` is single-quoted in `.env`.
- [ ] Each process's own ready line was confirmed in its log before driving
      it — no request fired against a not-yet-listening port.
- [ ] The full CRUD cycle (create → detail → edit → delete → re-fetch) ran
      for both Triggers and Templates, not just one.
- [ ] At least one negative case (no session, bad service-token body, a
      resource that no longer exists) was checked, not just the happy path.

## Out of scope — do not do these

- Redesigning the pattern itself, or deciding a new resource's descriptor
  shape — `zanix-remote-api-app-pattern`'s job.
- Adding a new resource/feature to console — `zanix-feature-builder`'s job,
  once this runbook (or the app already being up) confirms the app itself
  is healthy.
- Persisting this run as an automated test. Every automated test in this
  area deliberately mocks the hub-client factories for speed/determinism —
  that's a real, separate tradeoff, not an oversight this runbook should
  try to correct by proposing a real-network CI test.
- Fixing the known gaps listed above. Confirm whether each is still open,
  and file/track it — don't silently patch it as a side effect of running
  this runbook.

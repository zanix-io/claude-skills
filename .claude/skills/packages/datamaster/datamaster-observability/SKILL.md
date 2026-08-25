---
name: datamaster-observability
description: @zanix/datamaster's Elasticsearch/OpenSearch connector and the elasticsearchLogSave bridge for @zanix/logger. Use when wiring log/observability storage, choosing between one-time and persisted worker offload, or reviewing an ES/OpenSearch query.
---

For how the `'search'` slot registers itself, see
`datamaster-connector-registration`. File:line references point at
`~/Documents/Development/ZanixLibraries/datamaster` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- `search()`'s query is raw ES DSL, untyped by design — don't try to derive a
  typed wrapper from this skill; write the query directly against
  Elasticsearch/OpenSearch's own real API docs for the specific query shape
  needed.

## `ZanixElasticsearchConnector`

A plain `fetch`-based client, no vendor SDK — deliberately uses only "the most
stable, least product-differentiated part of the wire protocol" (`_doc`,
`_bulk`, `_cluster/health`) so the same connector works against ES OSS, ES Free,
and OpenSearch without a fork.

- **Auth**: Basic auth via URL userinfo (`https://user:pass@host:9200`). API-key
  auth has no URL equivalent, so it falls back to
  `ELASTICSEARCH_API_KEY`/`OPENSEARCH_API_KEY` env vars instead.
- **`ensureIndex()`** — HEAD-then-PUT-only-if-missing, **never overwrites an
  existing index**. Safe to call unconditionally on startup.
- **`search()`** is deliberately **untyped** on its `query` argument (raw ES
  DSL) — intentionally _not_ part of `@zanix/server`'s `ZanixSearchConnector`
  abstract base, since a typed abstraction over arbitrary ES DSL would either be
  incomplete or fight the real query language.

## `getConnector()` — no silent fallback construction

`getConnector()` (`observability/connector.ts`) resolves the shared `'search'`
core connector — it used to silently construct a standalone
`ZanixElasticsearchConnector` from bare env vars when nothing was registered;
**that fallback is gone.** If nothing is registered under `'search'`, it now
throws the real `@zanix/server` "did you forget to import
`@zanix/datamaster/core`?" error instead of masking a real misconfiguration
behind a connector that might be silently pointed at nothing real
(`http://localhost:9200` if even the bare env vars were unset).

**`elasticsearchLogSave`'s own `flushInline` wraps this in a try/catch**,
reporting a `getConnector()` throw the same way it reports a `bulkIndex()`
rejection — `elasticsearchLogSave` is a fire-and-forget `SaveDataFunction` with
a "never throws" contract of its own, so a caller of `save()` never sees this.
If you're writing a NEW consumer of `getConnector()` outside this bridge, decide
deliberately whether your own call site needs the same catch-and-report
wrapping, or whether letting the throw propagate is actually what you want
there.

**`flushBulkInWorker` (`worker-flush.ts`) never calls `getConnector()` at all**
— it constructs `new ZanixElasticsearchConnector(connectorOptions)` directly,
since a worker thread's own `ProgramModule` state starts fresh every time
(nothing from the main thread's registry survives the `postMessage` boundary);
checking the registry from inside a worker can never succeed, so
`getConnector()`'s new throw-on-empty behavior would fire on literally every
call there for a reason that isn't a real misconfiguration. If you add a new
worker-dispatched flush path, follow this same direct-construction pattern —
don't route it through `getConnector()`.

## `elasticsearchLogSave` — the `@zanix/logger` bridge

Factory returning a `SaveDataFunction` for `@zanix/logger`'s `storage.save`.
Timestamp field aliasing: watch for `@timestamp` vs `timestamp` — the two aren't
interchangeable, confirm which one the target index actually expects before
assuming a log write lands on the right field.

`useWorker: 'one-time' | 'persisted'` — `'persisted'` needs a booted Zanix Core
app; **outside one, it transparently falls back to `'one-time'`** rather than
failing. Don't assume `'persisted'` guarantees persistent worker offload in
every context — verify a Zanix Core app is actually running if that guarantee
matters for the deployment in question.

## `MeilisearchConnector` — the second real `'search'` engine, and why it needed real new code

Confirmed real, shipped precedent (`observability/meilisearch-connector.ts`) for
the case `datamaster-builder`'s own "S3 vs. search" note describes: unlike
`S3ObjectStorage` (already a generic client — a new S3-compatible backend is a
config change), a genuinely different search engine needs a real new
`ZanixSearchConnector` implementation, since `ZanixElasticsearchConnector`'s
wire protocol is ES/OpenSearch-specific.

The concrete difference that mattered most: **Meilisearch's document-write API
is asynchronous** (every write enqueues a task, `GET /tasks/{taskUid}` polled to
a terminal status) — a genuinely different completion-semantics shape from
Elasticsearch's synchronous-ish `_bulk`, not just a different dialect of the
same protocol. `bulkIndex()` polls by default (`waitForTask: true`) so its
`{errors, failedCount}` result reflects what actually happened, and a poll
timeout throws rather than guessing an outcome. **Before assuming a new search
backend's protocol is "just a different flavor of ES's shape," check whether its
writes are synchronous or an async job queue** — this is exactly the kind of
protocol-shape difference worth deciding explicitly, not inheriting from the
nearest precedent uncritically.

Also: Meilisearch has no separate single-vs-bulk endpoint (`index()`/
`bulkIndex()` both call the same `POST /indexes/{index}/documents`), and no
`ensureIndex()`-equivalent (the index auto-creates on first write) — real,
confirmed-against-Meilisearch's-own-docs differences, not assumptions.

**`'search'` is a single core-connector slot, confirmed NOT
independently-coexisting like `auth`'s OAuth2 providers** — verified empirically
(two classes each decorated `@Connector({slot: 'search'})` leaves only the
SECOND one registered, no error, no warning). **Superseded (2026-08-20)**:
this used to be guarded by `assertSearchConfigNotConflicting()` watching the
old per-backend `ELASTICSEARCH_URL`/`OPENSEARCH_URL`/`MEILISEARCH_URL` vars —
that guard is gone entirely. The real current mechanism is a single selector:
`SEARCH_ENGINE=elasticsearch|opensearch|meilisearch` + generic `SEARCH_URL`,
resolved via `resolveSearchEngine()` (`observability/search-config.ts`,
throws `InternalError`/`SEARCH_ENGINE_UNSUPPORTED` on an unrecognized value).
Two backends configured for the same deployment can no longer be
REPRESENTED, let alone silently collide — see `zanix-envvar-conventions` for
the general pattern this follows and `datamaster-connector-registration` for
how it plugs into a new mutually-exclusive slot.

## Zero-config registration

`./core` auto-registers under the `'search'` slot, gated on `SEARCH_ENGINE`
resolving to `'elasticsearch'`/`'opensearch'` (selects
`ZanixElasticsearchConnector`) or `'meilisearch'` (selects
`MeilisearchConnector`); `SEARCH_URL` is read by whichever class gets
selected (see `datamaster-connector-registration` for the general
unconditional-slot-vs-conditional-instance pattern this follows).

## Testing

Unit tests mock `fetch`. Both search engines have the same two-layer
coverage, and both suites pass together against real OpenSearch and
Meilisearch instances running simultaneously with no collision on the
conflict guard (neither functional file sets `ELASTICSEARCH_URL`/
`OPENSEARCH_URL`/`MEILISEARCH_URL` globally; both connectors default to
`localhost:9200`/`localhost:7700` directly). Both functional suites are
**skipped by default**, unlike the Mongo/Redis functional tests, which assume
that infra is always running locally and have no equivalent env-flag gate.

- **ES/OpenSearch** — split across two functional files:
  `src/@tests/functional/observability/connector-real.test.ts` (4 tests:
  cluster health, index/bulkIndex+search, a genuine mapping-conflict partial
  failure, `elasticsearchLogSave`) and `core.test.ts` (4 tests, the
  registration-DSL behavior itself). Only `connector-real.test.ts` is gated
  by `RUN_OPENSEARCH_TESTS=true` — it's the one making real network calls;
  `core.test.ts` runs unconditionally, since it only exercises the
  registration DSL and never touches a live cluster.
  Docker: `opensearchproject/opensearch:2`, security plugin disabled, ports
  9200/9600.
- **Meilisearch** — `src/@tests/functional/observability/meilisearch-connector-real.test.ts`,
  gated by `RUN_MEILISEARCH_TESTS=true`. Docker: `getmeili/meilisearch:v1.11`,
  port 7700. This suite is what surfaced a real, confirmed-against-a-live-instance
  fact the unit mocks didn't: a Meilisearch write task fails **atomically**
  (`details.indexedDocuments: 0` for the WHOLE batch on any invalid document,
  even mixed with otherwise-valid ones) — not per-document the way ES's
  `_bulk` partial-failure test (in the sibling ES file) is. If you're
  extrapolating `bulkIndex()`'s `failedCount` semantics from the ES connector's
  shape, don't — verify per engine, this is exactly the kind of thing that
  looked "probably the same" until checked against a live instance.
- Both are registered as CI `services:` (see `.github/workflows/publish.yml`)
  — Meilisearch's image needs no extra args unlike SeaweedFS's, so a plain
  `services:` block (not a manual `docker run` step) works for it, same as
  OpenSearch.
- **Not closeable this way**: `region`'s "real AWS rejects a mismatch" claim
  (`datamaster-storage`) — tried both SeaweedFS and LocalStack community as
  local backends, neither enforces SigV4 region validation. That gap is
  different in kind from this one — it needs real AWS, not a better local
  container — don't conflate the two when deciding what's worth chasing next.

## Checklist before wiring observability storage

- [ ] Is the timestamp field name (`@timestamp` vs `timestamp`) confirmed
      against the target index's real mapping, not assumed?
- [ ] If `useWorker: 'persisted'` is set, is a booted Zanix Core app actually
      guaranteed present in this deployment — or is the silent fallback to
      `'one-time'` an acceptable/expected outcome here?
- [ ] Does `ensureIndex()` get called unconditionally on startup rather than
      only on a first-deploy code path — it's safe to call every time.
- [ ] For a new `getConnector()` call site: does it need the same
      catch-and-report wrapping `flushInline` uses (a "never throws" contract of
      its own), or is letting the "nothing registered" throw propagate the
      actually-correct behavior here? And is it running on the main thread
      (registry lookup makes sense) or inside a worker (construct
      `ZanixElasticsearchConnector` directly instead, per `flushBulkInWorker`'s
      own precedent)?

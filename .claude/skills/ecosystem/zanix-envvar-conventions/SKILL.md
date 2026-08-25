---
name: zanix-envvar-conventions
description: How to shape a NEW env var (or group of related env vars) correctly across the Zanix ecosystem — the four-pattern taxonomy (single selector / separate-vars-plus-guard / independent-coexisting / other) that decides whether a mode-selector is the right fix or actively wrong, the real X_MODE=a|b|c + resolver precedent, and the _ENV constant-export convention. Use whenever a change introduces a new env var, or whenever two-or-more existing env vars look like they might conflict. Does NOT own "is this documented" — that's feature-completeness-conventions/docs-readme-audit's job.
---

Grounded in a full 12-repo audit (2026-08-20, "Env Var Selector Audit") — not
guessed from the two cases fixed so far. Re-verify a specific repo's current
state before citing it as settled; this skill records the taxonomy and the
convention, not a permanent snapshot of every repo's env vars.

## Golden rule (token savings)

- **Classify before building.** The question is never "does this group have
  more than one var" — it's which of the four shapes below it actually is.
  Only one of them (B) needs a selector; forcing one onto pattern C removes
  real, used capability, not just tidies naming.
- **This skill doesn't own documentation completeness.** Whether a new env
  var is actually written up in `docs/`/README/JSDoc is
  `feature-completeness-conventions`'s gate (already mandatory for every
  change) — don't duplicate that check here. This skill only owns whether the
  env var's own SHAPE is correct.

## The four patterns — classify every var group against these

| Pattern | Shape | Needs a selector? |
| --- | --- | --- |
| **A** | Single selector — one var (or one var + complementary vars for the chosen mode) already decides the outcome. No invalid state representable. | Already correct, nothing to do. |
| **B** | Separate vars + guard — N vars, each implying a different backend/mode for ONE shared slot, reconciled after the fact by a throw or (worse) silent priority order. | **The actual target — fix this.** |
| **C** | Independent, coexisting — N vars gate N things that can all be true AT ONCE (different slots, or genuinely parallel instances — OAuth2 providers, cache backends). | No — unifying would be wrong; it removes real coexistence. |
| **D** | Other — delegated to another package, a generic pass-through, a config-to-env writer, or a prerequisite pair (both required together, not alternatives — e.g. SSL cert+key). | Case by case; usually not a selector problem at all. |

**Confirmed empirically: pattern B is rare.** Of ~25 var-groups classified
across the ecosystem, only 4 were genuine pattern B — and forcing the
selector shape onto something that's actually pattern C (multiple cache
backends coexisting, multiple OAuth2 providers, Mongo vs. SQLite as
different slots entirely) would break a real, used configuration. Don't
pattern-match "multiple vars, similar names" to pattern B on sight — confirm
whether the vars can genuinely both be true at once (C) or are strictly
alternatives for one shared thing (B) before deciding anything needs to
change.

A prerequisite pair (D) is a common miscategorization to watch for: `server`'s
`SSL_KEY_PATH`/`SSL_CERT_PATH` look B-shaped (two vars, "pick one config") but
are actually AND, not XOR — there's nothing to select between, the real bug
there is silent fallback on incomplete config, not a missing selector.

**Confirmed real, 2026-08-20 — this classification prevented the wrong fix.**
Correctly resolved as pattern D, not given a selector: incomplete or
unreadable TLS config now throws a real `InternalError` immediately, naming
the var, the path, and the exact cause (missing vs. unreadable vs. other) —
`readSslFile()` replacing the old silent `fileExists()`-then-read. Followed
`auth`'s `ipAllowlistGuard` and `server`'s own `compileRuntime` as the
loud-failure precedent. This is the general lesson, not just an SSL-specific
one: when a group is genuinely pattern D (or C), the fix is a loud guard on
the group's own invalid states, never a selector bolted on because the group
"has more than one var."

## The real fix for genuine pattern B — one selector, package-local resolver

When a var group is genuinely pattern B, the fix is a single mode-selector
env var (`X_MODE=a|b|c`) paired with one small resolver function — **ad hoc
per package, no shared cross-repo abstraction**. This is a standing decision,
not an oversight: a `@zanix/server`-level shared helper stays on the table
only if a THIRD real pattern-B case turns up genuinely identical in shape —
don't extract one speculatively from two data points, even though both
converged independently on the same resolver shape (confirmed: they did).

Two real, shipped precedents, confirmed identical in shape — copy either as
the literal template:

- **`@zanix/datamaster`** — `SEARCH_ENGINE_ENV`/`SEARCH_URL_ENV` +
  `resolveSearchEngine()` (`observability/search-config.ts`). Replaced the
  old `ELASTICSEARCH_URL`/`OPENSEARCH_URL`/`MEILISEARCH_URL` pairwise-guard
  design (`assertSearchConfigNotConflicting()`, now gone).
- **`@zanix/notifications`** — `TEMPLATES_BACKEND_ENV` +
  `templatesBackendMode()` (`modules/templates/provider.ts`). Replaced the
  old `assertTemplatesConfigNotConflicting()` mode-inference guard. Note this
  one also killed a THIRD, now-redundant var (`DATABASE_TEMPLATES`) as part of
  the same fix — keeping it alongside the new selector would have recreated
  exactly the "two vars can contradict each other" problem the selector
  exists to remove; a genuine pattern-B fix should eliminate every var that
  used to imply a mode, not just add the new selector on top.

Both share the identical shape: `resolve<X>()` returns `<Mode> | undefined`
(`undefined` = nothing configured, the old "no-op" state), throws a real
`InternalError` naming the invalid value and the supported set if set to
anything else. Match this shape for a new pattern-B fix, don't invent a
different resolver contract.

**A third real case, confirmed 2026-08-20, deliberately NOT identical in
shape — don't let it look like "three now, extract the helper."**
`@zanix/notifications`'s SMS (`SMS_PROVIDER=twilio|vonage`) and WhatsApp
(`WHATSAPP_PROVIDER=meta|twilio`) provider selection replaced two
`hasTwilioEnv()`/`hasVonageEnv()`-shaped silent-priority checks — but unlike
search/templates, the selector here is **optional, not mandatory**: it's
only required (and only throws `InternalError`) when BOTH providers' own
credential vars are configured at once — the genuinely ambiguous case. With
only one provider's credentials set, the pre-existing zero-config path keeps
working exactly as before, no selector needed. This is the correct call
specifically because, unlike search/templates, the credential vars
themselves (`TWILIO_ACCOUNT_SID` etc.) were never the ambiguity — only the
silent arbitration order was. **When building a new pattern-B fix, check
whether the underlying per-backend vars are themselves already unambiguous
in the single-configured case** — if so, a mandatory selector is more
migration cost than the problem needs; make it conditional on genuine
conflict instead, the way this one does. This variance is exactly why the
shared-helper extraction stays deferred — three real cases now, but not
three identical shapes.

**A real, confirmed implementation hazard when two sibling modules each want
their own `_ENV` constant with the same literal name**: building this fix
surfaced that `sms/defs.ts` and `whatsapp/defs.ts` both tried exporting an
identically-named constant, which broke `core.ts`'s own `export *` barrel
(`TS2308`, duplicate export) — resolved by keeping ONE of the two sibling
files as the canonical exporter and having the other import from it, rather
than each defining its own copy. Watch for this whenever two related but
separate modules in the same package both need a constant with the same
name (e.g. two provider-selector modules sharing a credential var name).

**Real downstream cost of a hard rename, confirmed twice**: both fixes above
were breaking renames of already-published env var names, not additive
changes. Before shipping one, grep every OTHER Zanix repo (not just the one
being changed) for an import of the symbol/constant being removed — confirmed
real: `@zanix/admin`'s `metadata.ts` and `@zanix/core`'s `setup.ts` both
import `DATABASE_TEMPLATES_ENV`/`isDatabaseTemplatesDisabled` from
`@zanix/notifications`, and `core`'s `setup.ts` also imports
`ELASTICSEARCH_URL_ENV`/`OPENSEARCH_URL_ENV` from
`@zanix/datamaster/observability` — both currently shielded only by pinning
an already-published JSR version, not the local hard-rename. A hard rename
needs its downstream consumers' migration coordinated into the SAME release
window as the publish, not left to break at the next one.

## Separate, orthogonal convention: export a named `_ENV` constant for every var

Whether an env var's literal name is exported as a constant
(`TEMPLATES_MODEL_ENV = 'TEMPLATES_MODEL_NAME'`) or just typed inline at every
call site is unrelated to the selector question above, but real and
worth enforcing on every new var regardless of its pattern — see
`naming-and-structure-conventions` for why (compile-time vs. silent-runtime
failure on a typo).

Confirmed current state, per repo (re-verify before citing as still
accurate): `datamaster` does this consistently everywhere; `notifications` is
mixed (templates has it, email/sms/whatsapp are inline literals);
`admin`/`core` partial or imports-only; **`auth`, `asyncmq`, `server` have
ZERO exported `_ENV` constants at all**, in any of the three. A new env var
added to any of those three repos is adding to an already-inconsistent
baseline — export the constant anyway; don't match the repo's current
majority pattern just because it's the majority.

## Separate, orthogonal convention: export a companion `is<X>ResourceEnabled()` boolean

Also unrelated to the four-pattern classification above, but real and worth applying
whenever a resource's presence is gated by a single env-var-derived condition (pattern
A, or a pattern-B `resolve<X>()` result compared against one specific mode): the owner
package should export a companion `is<X>ResourceEnabled(): boolean` (or, when the
selector is multi-valued, `is<X>ResourceEnabled(mode): boolean` taking the mode
explicitly) alongside the raw selector/disable-flag — so a downstream consumer gets a
direct, correctly-named yes/no answer instead of re-deriving one from the selector's
own value at every call site.

**Shipped, real precedent (2026-08-22)** — three instances across two owner packages,
consumed by a third:

- `@zanix/datamaster` (`modules/database/providers/mongo/connector/mod.ts`):
  `isTriggersResourceEnabled(): boolean` (`=> !isTriggersModelDisabled()`).
- `@zanix/datamaster` (`modules/dlq/dlq.model.ts`): `isDlqResourceEnabled(): boolean`
  (`=> !!Deno.env.get(DLQ_MODEL_ENV)`) — DLQ had no prior helper of any shape.
- `@zanix/notifications` (`modules/templates/provider.ts`):
  `isTemplatesResourceEnabled(mode: TemplatesBackendMode): boolean` (`=>
  templatesBackendMode() === mode`) — **parameterized**, unlike the other two, because
  "templates enabled" has no single meaning on its own (a `'remote'` deployment has
  templates fully configured too, just not `'local'`). This is the worked example for
  when NOT to default the parameter: the first draft hardcoded `'local'` and was
  correctly rejected as ambiguous for a future non-admin consumer.
- `@zanix/admin` (`src/modules/admin-resource-gates.ts`) consumes all three, as a thin
  delegation layer, not the logic owner — see `admin-composition-and-extension`'s "REST
  and operations/mcp share ONE gate" section for the full consumer story (why two
  independent internal surfaces share these functions, and the real bug that motivated
  it); not repeated here.

**When it's worth the refactor vs. when it isn't**: check whether the raw selector
already has more than one internal comparison site before adding the boolean AND
refactoring existing call sites to use it — real in `@zanix/notifications`'s
`provider.ts` (three separate `templatesBackendMode() === 'remote'` comparisons
collapsed to `isTemplatesResourceEnabled('remote')`), NOT needed in
`@zanix/datamaster` (`isTriggersModelDisabled()` had exactly one caller before and
after; the triggers connector's own `if (!this.triggersModel) return` reads an
already-resolved instance field, not the raw env var, so it's not a candidate either).
Don't force a refactor onto a selector that already has exactly one reader.

**A pattern-B mode-selector's boolean companion still follows this skill's OWN
checklist item above** ("does every real read of it go through the one
`resolve<X>()`-shaped function") — `is<X>ResourceEnabled(mode)` should itself call
`resolve<X>()`/its own selector function, never re-derive the raw env var independently.

## Checklist before adding a new env var (or touching an existing group)

- [ ] Classified against the four-pattern table above — not assumed from
      "there are multiple related vars" alone?
- [ ] If pattern B: does the fix add ONE selector + package-local resolver
      (matching `resolveSearchEngine()`/`templatesBackendMode()`'s shape),
      and does it remove EVERY var that used to imply a mode — not just add
      the new selector alongside the old ones?
- [ ] If pattern C: confirmed the vars genuinely coexist (not forced into a
      selector that would break real parallel configuration)?
- [ ] If pattern D (prerequisite pair, config-to-env writer, delegated
      pass-through): confirmed it's not secretly pattern B before writing it
      off as "not a selector problem"?
- [ ] Is a hard-rename's real downstream impact checked — grepped every OTHER
      Zanix repo for an import of the symbol being removed/renamed, not just
      the owning repo's own call sites?
- [ ] Does the new var have an exported `_ENV` constant, regardless of
      whether the repo's existing vars mostly do?
- [ ] If a selector already exists for this value, does every real read of it
      go through the one `resolve<X>()`-shaped function — never re-compared
      inline at each call site? Confirmed real drift already: `asyncmq`'s
      `ZANIX_WORKER_EXECUTION` is a genuinely correct 3-state selector, but
      read via duplicated inline string comparisons in 3 separate files
      instead of the one `isInternalProcess()`-style helper `worker/mod.ts`
      already has — not a pattern problem, but a real DRY gap worth catching
      before it spreads to a 4th call site.
- [ ] (Not this skill's job, but don't skip it) — is the new var actually
      documented per `feature-completeness-conventions`'s Docs gate?

## Out of scope — do not do these

- Extracting a shared cross-repo selector-resolver helper (e.g. into
  `@zanix/server`) from the two current precedents — that's explicitly
  premature per the standing decision above; wait for a genuine third case.
- Forcing a selector onto a confirmed pattern-C group (cache backends,
  OAuth2 providers, per-feature admin toggles, Mongo vs. SQLite) — that
  removes real capability, it isn't a style preference to override.
- Deciding whether a new env var is documented — that's
  `feature-completeness-conventions`/`docs-readme-audit`'s gate, already
  mandatory; this skill only checks the var's own shape.
- Judging remediation urgency/order across multiple findings — report what
  pattern each group is and whether it needs fixing; sequencing across a
  whole ecosystem sweep is a human call.

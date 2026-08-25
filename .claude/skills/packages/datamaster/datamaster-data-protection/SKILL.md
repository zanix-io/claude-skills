---
name: datamaster-data-protection
description: mask/encrypt/hash field-level protection, internal/private/protected access strategies, versioned keys, and key rotation in @zanix/datamaster. Use when a model field needs to be protected at rest, when reviewing whether a field's protection strategy is appropriate, or when rotating protection keys.
---

For where `dataProtectionGetter`/`dataAccessGetter` attach inside a model's
`definition`, see `datamaster-database-and-models`. For the equivalent
encryption-at-rest pattern applied to object storage rather than model fields
(and how its failure mode deliberately differs from this one), see
`datamaster-storage`. File:line references point at
`~/Documents/Development/ZanixLibraries/datamaster` — read the real code
there before assuming this summary is still accurate.

## Golden rule (token savings)

- The three protection strategies and three access strategies are a fixed,
  closed set (mask/encrypt/hash; internal/private/protected) — confirm which
  one a field needs against this skill's tables directly, don't re-derive
  the taxonomy from source each time.
- Verify a specific typing pattern (the `Required<Strategy>Scalar/Array` +
  `as` assertion convention) against one real example in the target repo,
  not by re-reading `data-protection.md` in full for every field added.

## Protection strategies (`dataProtectionGetter`)

Three strategies, applied via a getter on the field's schema definition:

| Strategy | Reversible? | Read back via |
| --- | --- | --- |
| `mask` | Yes, synchronously | `.unmask()` |
| `encrypt` | Yes, asynchronously | `.decrypt()` |
| `hash` | No — one-way | `.verify(input)` |

Every protected value comes back as a **boxed `String` object**, not a
primitive string — that's what carries the `.unmask()`/`.decrypt()`/
`.verify()` methods. Typing a hydrated protected field uses `Required<Strategy>Scalar`/
`Required<...>Array` types with an `as` assertion, **never a `:` type
annotation** — the getter's real runtime return type (the boxed wrapper) and
its TypeScript-declared type intentionally diverge, so declaring it with `:`
fights the actual shape instead of describing it.

## Access strategies (`dataAccessGetter`)

Independent of protection — controls whether a field is visible in a
client-facing read (`toJSON()`/`toObject()`) at all, based on whether the
caller is authenticated:

| Strategy | Anonymous read | Authenticated read |
| --- | --- | --- |
| `internal` | Removed | Removed — always, even authenticated |
| `private` | Removed | Shown |
| `protected` | Masked | Shown |

`dataPoliciesGetter` combines both concerns (protection + access) on one
field, but **does not run both policies on every read** — access control only
kicks in during `toJSON()`/`toObject()`; a raw document read (not serialized
through those) bypasses the access strategy entirely, though not the
protection strategy (protection is enforced at the schema/setter level,
independent of serialization).

## Protection timing

- **`pre('save')` fires only on first save** (`isNew`) — an update to an
  existing document does NOT re-run automatic protection through this hook
  alone. For explicit, one-off protection outside of a save, use the static
  methods directly: `Model.hash`/`encrypt`/`mask`/`unmask`/`validateHash`.
- **`extensions.autoProtectOnUpdate: true`** extends automatic protection to
  updates on existing documents too, via a snapshot-diff comparison — falls
  back to the `AUTO_PROTECT_ON_DB_UPDATE` env var when the option itself is
  left unset. Without this, a field changed via `.save()` on an existing doc
  silently skips protection unless you called the static method yourself —
  worth checking explicitly on any model whose protected fields get updated
  after creation, not just set once.

## Query-level protection (`useDataPolicies`)

Opt-in on both sides of a query:

- **Write side** — `updateOne`/`findOneAndUpdate`/`bulkWrite` accept
  `useDataPolicies: true` to protect the values being written, the same way
  a `.save()` would.
- **Read side** — `find`/`findOne`/`countDocuments`/`paginate` accept it to
  rewrite a **plaintext filter condition on a `mask`-strategy field** into
  its protected form before the query runs, so `{ filter: { taxId: '123' } }`
  actually matches the masked value stored in the database.
- **Throws on anything beyond an equality-shaped filter** on a protected
  field — `$regex`, `$gt`, and similar operators aren't rewritable against a
  masked/hashed/encrypted value, so the query throws rather than silently
  matching nothing or matching the wrong thing. **`hash`/`encrypt` strategies
  throw even on plain equality** — neither supports any query-time rewrite at
  all (hash has no reverse; encrypt would require decrypting every stored
  value to compare, which this package doesn't do).

## Versioned keys and rotation

Every protection key (`DATA_SECRET_KEY`/`DATA_AES_KEY`/`DATA_RSA_PUB`/
`DATA_RSA_KEY`) supports the ecosystem's versioned-env-var convention:
append `_V1`, `_V2`, etc. to the base name for a specific version; the
**unsuffixed** variable is the implicit `v0` default and never gets a
suffix. Config shape is `{ activeVersion, versionConfigs }` — **reading**
auto-detects the version from the stored value's own prefix (falling back to
`versionConfigs.default` if the prefix is missing/unrecognized); **writing**
always uses `activeVersion`, regardless of what version any existing stored
value happens to be at.

```ts
seedRotateProtectionKeys() // unconditionally re-protects every document, every run
const { total, current, outdated } = await checkProtectionRotationStatus(Model) // per-path report
```

`checkProtectionRotationStatus` is the read-only report; `seedRotateProtectionKeys`
is the actual migration. Run the check first — `seedRotateProtectionKeys`
re-protects unconditionally, so confirm there's real `outdated` count before
running it against a large collection for no reason. The identical
check/rotate pairing pattern reappears in `datamaster-storage` for
object-encryption keys — same shape, different data.

## Two caution points — don't transcribe these as unqualified best practice

- **`DATA_SECRET_KEY` is checked first for masking, falling back to
  `DATA_AES_KEY`** if unset — this fallback exists for convenience, but means
  a deployment that only sets `DATA_AES_KEY` gets masking silently backed by
  a key meant for a different strategy. Confirm which key is actually in use
  in a given deployment rather than assuming `DATA_SECRET_KEY` is always set.
- **Encryption strategy failure mode is fail-open in this package** (see
  `datamaster-dlq`'s payload protection, which shares this failure mode) —
  contrast this explicitly with `datamaster-storage`'s object-encryption,
  which **throws** on a missing/invalid key instead. Don't assume "encrypt"
  means "guaranteed encrypted, or the app refuses to start" everywhere in
  this package — verify per feature which failure mode applies before relying
  on it for a real security property.

## Checklist before protecting a new field

- [ ] Chosen strategy matches the real need: `mask` for something searchable/
      partially visible, `encrypt` for something that needs full plaintext
      recovery, `hash` for something that only ever needs verification
      (passwords, tokens)?
- [ ] Chosen access level matches who should ever see it: `internal` if truly
      no client should ever see it (even authenticated), `private` if only
      authenticated callers should, `protected` if anonymous callers get a
      masked view?
- [ ] Does this field get updated after creation? If so, is
      `autoProtectOnUpdate` on (or the static method called explicitly) —
      otherwise a later `.save()` on an existing doc silently skips
      protection?
- [ ] Any query filtering on this field goes through `useDataPolicies: true`,
      and only uses equality-shaped conditions if the field is `mask`-strategy
      (never `hash`/`encrypt`, which can't be queried at all)?
- [ ] Is the protection key version scheme (`_V1`/`_V2`/unsuffixed `v0`) being
      followed consistently, so a future rotation has a real version history
      to walk?

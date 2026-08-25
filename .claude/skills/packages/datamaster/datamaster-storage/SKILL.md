---
name: datamaster-storage
description: @zanix/datamaster's object storage (S3-compatible, e.g. SeaweedFS) and file-metadata registry — two independent, domain-agnostic modules, plus encryption-at-rest and key rotation. Use when storing/retrieving a file's bytes or metadata, or reviewing object-storage encryption configuration.
---

This is the newest area of the package (`./storage`/`./files` subpaths, added
recently — verify against the real `CHANGELOG.md`/`deno.jsonc` version before
treating any detail here as durable; it's the doc most likely to have moved
since this skill was written). For the equivalent field-level protection pattern
applied to model data instead of object bytes, see `datamaster-data-protection`
— the rotation-status/rotate pairing is the same shape in both. File:line
references point at `~/Documents/Development/ZanixLibraries/datamaster` — read
the real code there before assuming this summary is still accurate.

## Golden rule (token savings)

- The two modules (`storage`, `files`) are independent — read only the one
  relevant to the task (bytes vs. metadata), not both, unless the task genuinely
  spans both.
- Confirm the rotation-status numbers with `checkEncryptionRotationStatus()`
  before running `rotateEncryptionKeys()` against a large bucket for no reason —
  same discipline as `datamaster-data-protection`'s key rotation.

## Two independent modules, neither domain-aware

- **`./storage`** — `S3ObjectStorage` (renamed from `SeaweedFSObjectStorage` —
  it was never actually SeaweedFS-specific, just branded after one particular
  self-hosted S3-compatible deployment; see `CHANGELOG.md` for the full rename),
  a byte store wrapping the real `@aws-sdk/client-s3` `S3Client` with
  `forcePathStyle: true`, registered under the `'s3'` core connector slot (see
  `datamaster-connector-registration` — `'s3'` isn't one of `@zanix/server`'s
  six hardcoded slots, so resolve via `this.connectors.get('s3')`, no dedicated
  getter). **`region` is a real, overridable option** (falls back to
  `S3_REGION`, then a harmless `DUMMY_REGION` dummy) — fixed after being
  empirically confirmed missing entirely: harmless against a self-hosted gateway
  that never validates region, but a real SigV4 signature failure against
  genuine, non-`us-east-1` AWS S3. Don't assume "genuinely generic S3 client"
  covers every field just because most do — confirm empirically like this case
  did, don't take the claim on faith for a new option. **The "real AWS would
  reject a wrong region" half of that claim is structurally unverifiable
  locally — confirmed by actually trying**: SeaweedFS AND LocalStack (community
  edition, `localstack/localstack:3.8`, tested directly against `S3ObjectStorage`
  with three different regions against the same bucket) both accepted every
  region tried, correct or fabricated, with no rejection. Don't reach for
  LocalStack expecting it to close this specific gap — it doesn't, at least not
  the community edition. What running against LocalStack DOES add over the unit
  test: proof `region` flows through the real `@aws-sdk/client-s3` SigV4 signing
  pipeline against a genuine network call, not just that `client.config.region()`
  resolves correctly against a stub — a real but narrower guarantee than "AWS
  itself would reject a mismatch," which remains provable only against real AWS
  with real credentials. The reusable technique for what CAN be verified locally
  lives in `src/@tests/unit/storage/s3-object-storage-generic-backend.test.ts`:
  stub `S3Client.prototype.send` and assert directly on `this.config` inside
  the stub, rather than trying to infer the real client config from
  `S3ObjectStorage`'s own public surface — copy this file's approach for any
  new option added to this connector, don't reinvent the stubbing from scratch.
- **`./files`** — `MongoFileRepository`, a metadata registry. Follows the same
  `@Provider`/`ZanixProvider` shape as `TriggersAdminRepository`/ `DlqProvider`
  — caller-assigned id persisted as the document's native `_id`.

**Neither module has any notion of a specific domain, file kind, or processing
state** — that's deliberate; both are generic building blocks a consumer (e.g.
`@zanix/core`'s `AssetService`) composes into something domain-specific, not
something this package itself understands.

## Encryption at rest

Two modes:

- **`'symmetric'`** — AES-GCM, direct.
- **`'asymmetric'`** — RSA-wrapped per-object AES key, the wrapped key stored as
  S3 object metadata (`x-amz-meta-wrapped-key`).

`encrypt: false` **explicitly set** is a real, distinct state from **omitting**
`encrypt` entirely (which lets the env var apply instead) — an explicit `false`
always disables encryption regardless of any env var; leaving it unset is what
makes the env var take effect.

**Caution, don't transcribe this as matching every other package's convention**:
a missing or invalid encryption key here **throws** — this is the opposite
failure mode from `datamaster-data-protection`'s field-level `encrypt` and
`datamaster-dlq`'s payload protection, both of which fail open (silently store
plaintext). The docs contrast this explicitly. If you're moving between these
three protection mechanisms in the same session, don't assume they share a
failure mode — verify per mechanism.

### Key rotation

Same shape as `datamaster-data-protection`'s: `encrypt.version` is recorded
per-object as S3 metadata (`x-amz-meta-encryption-version`).

```ts
const status = await checkEncryptionRotationStatus(storage); // reporting, read-only
await rotateEncryptionKeys(storage); // the actual migration
```

Uses the same `_V1`/`_V2`/unsuffixed-`v0` versioned-env-var convention as
`datamaster-data-protection` — reusing the same env vars, not a parallel set.

## Local fallback combinators

- `createLocalFilesystemObjectStorage` — a filesystem-backed implementation of
  the same `ObjectStorage` interface, for local dev/testing or a no-S3-available
  deployment.
- `createFallbackObjectStorage` — wraps a primary + fallback pair, with
  `ensureSynced` reconciliation between them.
- `ensureLocalObjectsSynced` — the reconciliation helper itself.

These are what `@zanix/core`'s `AssetService` composes on top of, not something
most consumers reach for directly unless building an equivalent composition
layer.

**Security note, worth checking explicitly**:
`createLocalFilesystemObjectStorage` resolves a caller-supplied key to a real
filesystem path — confirm the version in use actually confines that path (a
`confinePath`-style guard against `../`/absolute-path escapes), rather than
assuming any key is safe to join onto the root directory unchecked. This is
exactly the shape of bug a real security audit of this ecosystem already flagged
as a path-traversal risk in a local filesystem storage backend — verify the fix
is present in the version you're on rather than assuming it by default.

## Testing

Functional suite gated by `RUN_S3_TESTS=true` (renamed from
`RUN_SEAWEEDFS_TESTS`) — **does not check whether the real backend is actually
reachable**; enabling the flag is a commitment to having the real integration
available, not something the test suite verifies for you and skips gracefully if
absent. Docker: `chrislusf/seaweedfs server -s3` on port 8333; CI creates the
test bucket via a `PUT http://localhost:8333/zanix-objects` step before running
tests, since a fresh SeaweedFS instance starts with no buckets.

## Checklist before adding a new storage-backed feature

- [ ] Does the feature actually need `./storage` (bytes), `./files` (metadata),
      or both — not defaulting to both out of uncertainty?
- [ ] Is encryption's failure mode (throws, unlike the rest of the package)
      accounted for — a missing key here is a hard failure, not a silent
      plaintext fallback?
- [ ] Any caller-supplied key/path confined against traversal before being
      joined onto a filesystem or bucket path?
- [ ] If rotating keys, was `checkEncryptionRotationStatus()` run first to
      confirm there's real work to do?
- [ ] Since this is the newest and fastest-moving area of the package — has
      the claim actually being cited here been re-checked against current
      source, not assumed stable without re-checking?

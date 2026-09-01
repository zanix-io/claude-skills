---
name: utils-encryption-and-masking
description: AES/RSA/HMAC symmetric and asymmetric encryption, salted unidirectional hashing, and reversible/irreversible masking — including real defaults worth double-checking (generateAESKey's 128-bit default, HMAC's two functions with different default hash algorithms) and unmask's silent-warning-not-throw behavior on 'hard' masks. Use when encrypting/decrypting, hashing, signing, or masking a value.
---

Covers `@zanix/utils/helpers`'s encryption/masking primitives — the layer
`@zanix/datamaster`'s data-protection and storage encryption both build on.
File:line references point at
`~/Documents/Development/ZanixLibraries/utils` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- Check the real default for whichever function is in play before assuming
  a "secure by default" posture — several defaults here are intentionally
  the *weakest* supported option (`generateAESKey`'s 128-bit default), not
  the strongest.
- `encrypt`/`decrypt` auto-detect RSA by PEM-prefix string matching — pass
  `type` explicitly when there's any doubt about which path a given key
  will route through.

## Symmetric: AES-GCM

```ts
const key = await generateAESKey() // 128-bit by default — see caution below
const encrypted = await encryptAES('hello world', key)
const decrypted = await decryptAES(encrypted, key)
```

`generateAESKey(length?)` — `length: 128 | 192 | 256`, **default `128`**,
the weakest of the three supported lengths — don't assume this call
defaults to the strongest option; pass `256` explicitly when that's the
actual requirement. `generateCustomAESKey(secret, toString?)` derives a
key from an arbitrary secret via SHA-256, padded to a valid AES length
(`toString` default `true` returns base64; `false` returns `Uint8Array`).
`encryptAES(message, key, ivLength?)` — `ivLength: 12 | 16`, default `12`.

**Generic routing**: `encrypt(message, key, type?)`/`decrypt(encryptedMessage,
key, type?)` route to AES unless `type === 'RSA'` or `key` starts with
`-----BEGIN` (PEM) — in which case they route to `encryptRSA`/`decryptRSA`.
This string-prefix auto-detection is convenient but easy to get wrong
silently: a non-PEM string accidentally passed where an RSA key was
intended routes through AES instead, with no error pointing at the
mismatch — pass `type` explicitly whenever the key's shape isn't
guaranteed at the call site.

## Asymmetric: RSA-OAEP (encryption) / RSASSA-PKCS1-v1_5 (signing)

```ts
const { privateKey, publicKey } = await generateRSAKeys() // 2048-bit, SHA-256 by default
const encrypted = await encryptRSA(message, publicKey)
const decrypted = await decryptRSA(encrypted, privateKey)

const signature = await signRSA(message, privateKey)
const valid = await verifyRSA(message, signature, publicKey)
```

`generateRSAKeys(options?)` — `hash` (excludes `'SHA-1'`) default
`'SHA-256'`; `modulusLength` default `2048`; `algorithm`
(`'RSA-OAEP' | 'RSA-PSS' | 'RSASSA-PKCS1-v1_5'`) default `'RSA-OAEP'`, and
only selects the generated keypair's WebCrypto `keyUsages`
(`'RSA-OAEP'` → encrypt/decrypt, anything else → sign/verify) — it has no
effect on what `signRSA`/`verifyRSA` actually do below.

**`signRSA`/`verifyRSA` always sign and verify with `RSASSA-PKCS1-v1_5`**,
regardless of which `algorithm` generated the keypair — this is what
"RS256"/"RS384"/"RS512" actually mean per RFC 7518 §3.3, distinct from
RSA-PSS (RFC 7518 §3.5, "PS256"/etc). A signature from `signRSA` verifies
under any spec-compliant external RS256 verifier (PyJWT, `jose`,
`jsonwebtoken`, `openssl`), not just this package's own `verifyRSA` — this
backs `@zanix/auth`'s JWT `RS256` signing (`createJWT`/
`createServiceAssertion`).

**HMAC — two functions, different default hash, easy to conflate**:
`signHMAC(data, secret, hash?)` is string-based, `hash` excludes `'SHA-1'`
(JWT has no HS1), default `'SHA-256'`. `signHMACBytes(key, data, hash?)`
takes raw `Uint8Array` key/data instead of strings, accepts the **full**
`HashAlgorithm` range including `'SHA-1'`, and **defaults to `'SHA-1'`**
— use this variant for arbitrary binary secrets (e.g. TOTP) to avoid a
string round-trip corrupting the key, but don't assume it shares
`signHMAC`'s default hash.

## Unidirectional hashing

| `level` | Algorithm | Iterations |
| --- | --- | --- |
| `'low'` | SHA-1 | 1000 |
| `'medium'` (default) | SHA-256 | 5000 |
| `'medium-high'` | SHA-384 | 8000 |
| `'high'` | SHA-512 | 10000 |

```ts
const hash = await generateHash(message) // 'medium' by default
const isValid = await validateHash(inputMessage, hash) // reuses the embedded salt
```

`useSalt` accepts a `Uint8Array`, a number (salt length in bytes, default
`16`), or `false` to disable — three different accepted types for one
parameter, worth double-checking which form is actually being passed. The
salt is prefixed to the returned hash (`"<salt-hex>$<base64-hash>"`) and
`validateHash` reuses it automatically.

## Masking

```ts
const masked = mask('4111 1111 1111 1234', 'my-secret', { startAfter: 0, endBefore: 4 })
const original = unmask(masked, 'my-secret', { endBefore: 4 }) // 'xor' only, reversible

const hardMasked = mask('4111 1111 1111 1234', '*', { algorithm: 'hard', endBefore: 4 })
// "**************** 1234"
```

`mask(input, mask, options?)` — `options.algorithm: 'xor' | 'hard'`,
default `'xor'`; `startAfter`/`endBefore` (a char count, or a character to
search for) leave a prefix/suffix unmasked.

**`'hard'` masking is irreversible by design, and `unmask` doesn't fail
loudly on it** — calling `unmask` on hard-masked data silently returns the
input unchanged and only logs a warning, never throws. The public
`UnMaskingOptions` type only accepts `'xor'` for `algorithm`, but a caller
passing already-hard-masked data still gets this silent-no-op behavior
rather than a type-level or runtime error catching the mistake early.

**Real footgun**: if the `mask` value passed to the `'hard'` algorithm is
longer than one character, only the first character is used — silently, with
a logged warning, not an error.

## Checklist before adding/reviewing an encryption or masking call

- [ ] Is `generateAESKey`'s length explicit (`256` for anything beyond a
      low-stakes use) rather than relying on its 128-bit default?
- [ ] Is `type` passed explicitly to `encrypt`/`decrypt` whenever the key's
      PEM-vs-non-PEM shape isn't guaranteed at the call site?
- [ ] For HMAC, is the right function (`signHMAC` for strings, default
      SHA-256; `signHMACBytes` for raw bytes, default SHA-1) used with its
      actual default hash understood, not assumed uniform across both?
- [ ] Is `'hard'` masking used only where the value genuinely never needs
      to be recovered — since `unmask` won't error to catch a mistaken
      assumption otherwise?

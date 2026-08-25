---
name: utils-validator-decorators
description: The full validation decorator catalog (strings, numbers, dates, arrays, enum, boolean) and their raw predicate counterparts — including real per-decorator gotchas like ArrayLength's min:2 default rejecting empty-allowed arrays, and IsNumberString never converting. Use when picking or reviewing a specific validation decorator on an RTO field.
---

Covers the concrete decorator catalog built on `utils-validator-core`'s
mechanics — read that skill first for `expose`/`optional`/`transform` and
`classValidation` itself. File:line references point at
`~/Documents/Development/ZanixLibraries/utils` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- Check this catalog's table before assuming a decorator's default
  behavior — several defaults are genuinely surprising (`ArrayLength`'s
  `min: 2`, string-typed number/boolean decorators that never convert).
- Every decorator here has a raw, lowercase predicate counterpart
  (`isString`, `isEmail`, ...) for validating a plain value outside a
  `BaseRTO` — reach for those directly when there's no RTO involved.
- `IsString`/`IsNumber`/`IsEnum`/`IsArray`/`IsBoolean`/`Expose` also each
  register a class-level `RTOFieldMetadata` entry (decorator name + args) —
  see `classMetadata` in `utils-validator-core` for introspecting a `BaseRTO`
  class's field shape with no instance/payload involved.

## Strings

| Decorator | Regex/rule | Notes |
| --- | --- | --- |
| `IsString` | — | Plain string check. |
| `IsEmail` | `EMAIL_REGEX` | |
| `IsUrl` | `URL_REGEX` | Scheme + optional `www.`. |
| `IsUUID` | `UUID_REGEX` | Validates UUID **v4** specifically. |
| `IsObjectID` | `OBJECT_ID_REGEX` | 24-character hex string (MongoDB `ObjectId`). Predicate is `isObjectId`/`isObjectIdArray` (lowercase `d`) — only the decorator and the exported constant name are capitalized `ID`. |
| `IsPhone` | `PHONE_REGEX` | E.164-like: optional `+`, 2–15 digits, first digit non-zero — **spaces not allowed**, `'+1 234 567 890'` fails. |
| `IsNumberString` | `^\d+(\.\d+)?$` | **Does not convert to a number** — value stays a string. Easy to confuse with `IsNumber`. |
| `IsBooleanString` | `"true"`/`"false"`, case-insensitive | **Stays a string** — does not convert to a real boolean. |
| `Length({min?, max?}, options?)` | — | `min` default `0`, `max` default `Infinity`. |
| `Match(pattern, options?)` | custom `RegExp` | |

Each has an array variant (`isEmailArray`, `stringLengthArray`, etc.) and a
raw lowercase predicate (`isString`, `isEmail`, ...).

## Numbers and dates (`DefaultTransformValidationOpts`)

| Decorator | Default `transform` | Self-exposes |
| --- | --- | --- |
| `IsNumber(options?)` | `true` — converts via `Number(...)` | yes |
| `MinNumber(num, options?)` | `true` | yes |
| `MaxNumber(num, options?)` | `true` | yes |
| `IsDate(options?)` | `true` — converts via `new Date(...)`, rejects `Invalid Date` | yes |
| `MinDate(date, options?)` | `true` | yes |
| `MaxDate(date, options?)` | `true` | yes |

Pass `{transform: false}` on any of these to keep the raw input type
instead of coercing — see `utils-validator-core` for the general
`expose`/`transform` mechanics these all follow.

## Arrays

```ts
@ArrayLength({ min: 1 }) // NOT { min: 0 } — see the real footgun below
@IsString({ each: true, expose: true })
accessor tags!: string[]
```

- `IsArray(options?)` — with `each: true`, validates array-of-arrays;
  `isArrayOfArray` (the raw predicate) wraps a non-array top-level value in
  `[value]` before checking, so it simply fails as `false` rather than
  throwing.
- `ArrayLength({min?, max?}, options?)` — **`min` defaults to `2`, not
  `0`**. **Real, explicitly-documented footgun**: the underlying raw
  predicate internally requires `min >= 1` — passing `min: 0` to express
  "any length, including empty" doesn't work; it makes the check **always
  return `false`**, even for a genuinely non-empty array. Always pass an
  explicit `min: 1` (or higher) — `{min: 0}` is never a valid way to allow
  an empty array. `ArrayLength` doesn't support `each`.

## Enum and boolean

```ts
const ROLES = ['admin', 'editor', 'viewer'] as const
@IsEnum(ROLES as unknown as string[], { expose: true })
accessor role!: string
```

- `IsEnum(validations: EnumType, options?)` — `EnumType = Record<string,
  unknown> | unknown[]`, typed as a **mutable** array; a `readonly`/`const`
  array needs an `as unknown as string[]` cast to satisfy the type.
  `isEnumArray` (the raw predicate) returns `false` — not a thrown error —
  when `value` isn't an array at all.
- `IsBoolean(options?)` — strictly `true`/`false`, no string coercion (see
  `IsBooleanString` above for the string-typed variant).

## `Expose`: exposing with no validation logic

```ts
@Expose()
accessor metadata?: Record<string, unknown>
```

`Expose(options?)` — a decorator with no validation function at all, for a
field that should just pass through onto the instance. Accepts
`message`/`optional`/`transform` (not `each`/`expose` itself, since
exposing is its entire purpose).

## Checklist before choosing/reviewing a decorator

- [ ] For `ArrayLength`, is `min` an explicit `1` or higher — never `0` to
      mean "allow empty," which silently rejects every array instead?
- [ ] Is `IsNumberString`/`IsBooleanString` used deliberately (value stays
      a string) rather than confused with `IsNumber`/`IsBoolean` (which
      convert)?
- [ ] For `IsEnum` with a `readonly`/`const` array, is the
      `as unknown as string[]` cast applied — or does the type error need
      resolving?
- [ ] Does a nested-array-with-non-array-input case actually need to
      fail as `false` (the real `isArrayOfArray` behavior), or does the
      calling code assume it throws?

---
name: utils-validator-core
description: BaseRTO, classValidation, classMetadata (class-level field introspection with no instance required), defineValidationDecorator, and the expose/transform/optional mechanics every validation decorator builds on — the most-consumed module in the ecosystem, since every Zanix package handling incoming data builds on it. Use when authoring an RTO class, calling classValidation directly, building a custom validation decorator, or needing a class's field/decorator shape at build time (e.g. an OpenAPI/schema generator).
---

Covers the core mechanics of `@zanix/utils/validator` — the accessor-decorator
validation system every `BaseRTO` subclass uses. For the full decorator
catalog (strings/numbers/dates/arrays/enum) and their individual gotchas,
see `utils-validator-decorators`. File:line references point at
`~/Documents/Development/ZanixLibraries/utils` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- In a Zanix HTTP framework (`@zanix/server`), you rarely call
  `classValidation` yourself — declare the RTO on the route/handler config
  (`Body`/`Params`/`Search`) and the framework calls it internally. Reach
  for calling it directly only outside that flow (a message-queue job, a
  CLI argument, standalone use).
- Every validated field must be `accessor`, never a plain class field —
  check this first before debugging why a decorator "isn't working."

## `BaseRTO` and `classValidation`

```ts
import { BaseRTO, classValidation, IsEmail, IsNumber, IsString, MinNumber } from 'jsr:@zanix/utils@[version]/validator'

class UserRTO extends BaseRTO {
  @IsString({ expose: true })
  accessor name!: string

  @IsEmail({ expose: true })
  accessor email!: string

  @IsNumber()
  @MinNumber(18)
  accessor age!: number
}

const user = await classValidation(UserRTO, { name: 'Ana', email: 'ana@example.com', age: '30' })
```

`classValidation(RTO, plainObject, options?)` returns
`Promise<InstanceType<RTO>>`, rejecting on validation errors.

| Option | Type | Default | Effect |
| --- | --- | --- | --- |
| `ctx` | `any` | `{}` | Injected, readable as `this.context` in constructors/custom validators/messages. |
| `excludeExtraneousValues` | `boolean` | `true` | `false` copies properties not assigned by the RTO as-is. |
| `exposeDefaultsValues` | `boolean` | `true` | `false` stops exposed accessors falling back to their default when the payload omits the property. |
| `exposeValuesAsGetter` | `boolean` | `false` | Expose resolved values as getters instead of plain properties — **not recommended with nested objects**, can interfere with serialization. |
| `throwErrors` | `(errors) => void` | throws `HttpError('BAD_REQUEST', ...)` | Custom handler replacing the default throw. |

Default thrown error's `cause`: `{ message: 'Request validation error',
target: RTO.name, properties: <formatted errors> }` — each
`properties[prop]` is `[{ constraints: string[], value, plainValue }]`,
nested errors surface as `{ message, properties }` under the parent key.

## `expose`, `optional`, `transform`: the three switches every decorator shares

**`expose`**: a decorator like `IsString`/`IsEmail` still validates the
value but leaves the property **unset** on the instance unless `expose:
true` (or a separate `@Expose()`) is passed — a real silent-no-op trap if
forgotten, since validation still runs and doesn't error. `IsNumber`,
`MinNumber`, `MaxNumber`, `IsDate`, `MinDate`, `MaxDate`, `ValidateNested`
self-expose automatically — no `expose: true` needed for those.

**`optional`**: skips validation entirely when the value is absent;
without it, a missing required field fails validation with a real error
(for `ValidateNested` specifically: `"The '<property>' property must be
defined."`).

**`transform`**: `IsNumber`/`MinNumber`/`MaxNumber` default `transform:
true` (converts via `Number(...)`); `IsDate`/`MinDate`/`MaxDate` default
`transform: true` (converts via `new Date(...)`, rejects `Invalid Date`).
Pass `{transform: false}` to keep the raw input type instead of coercing.

## `defineValidationDecorator`: the low-level primitive

```ts
import { defineValidationDecorator } from 'jsr:@zanix/utils@[version]/validator'

const IsPositive = defineValidationDecorator((value: number) => value > 0)
```

Turns a validation function into an accessor decorator — wires the setter
(runs on every assignment: applies `transform`, calls `validation`) and a
constructor-time `init` hook (applies `expose`/`optional`). Every built-in
`Is*`/`Min*`/`Max*` decorator is itself built on this. Use it directly only
when the higher-level `Validation()` decorator (below) isn't flexible
enough for a genuinely new validation shape — for a routine app-specific
rule, `Validation()` is the idiomatic choice (`utils-validator-decorators`).

Its optional third argument, `meta: { decorator: string; args?: unknown[] }`,
is what feeds `classMetadata` (below) — pass it when building a custom
decorator you want to show up there too; omit it and the field still
registers, just without a known `decorator` name.

## `classMetadata`: class-level field introspection, no instance required

```ts
import { classMetadata, IsEnum, IsString } from 'jsr:@zanix/utils@[version]/validator'

class RoleRTO extends BaseRTO {
  @IsString({ expose: true })
  accessor name!: string

  @IsEnum(['admin', 'user'], { expose: true })
  accessor role!: string
}

classMetadata(RoleRTO)
// {
//   name: { decorator: 'IsString', args: [], each: false, optional: false, expose: true },
//   role: { decorator: 'IsEnum', args: [['admin', 'user']], each: false, optional: false, expose: true },
// }
```

Where `classValidation` needs a plain object to validate, `classMetadata(RTO)`
answers "what fields does this `BaseRTO` have, with what decorator/args/
`each`/`optional`/`expose`" from the class definition alone — the piece a
build-time consumer (a schema/OpenAPI generator, a form/table renderer) needs
without constructing anything. Built on the native decorator-metadata
proposal (`Symbol.metadata`/`context.metadata`, real per this repo's Deno/TS
version — no `reflect-metadata`), not `ValidationsMetadata`
(`base/metadata.ts`), which is a completely separate, per-**instance**
runtime-validation-state tracker; don't confuse the two when reading the
source. Fields declared on a parent `BaseRTO` are included for a subclass
that extends it — a field the subclass redeclares overrides the parent's
entry.

Only `IsString`/`IsNumber`/`IsEnum`/`IsArray`/`IsBoolean`/`Expose` register a
known `decorator` name today (`utils-validator-decorators`'s full catalog —
`Length`/`MinNumber`/`MaxNumber`/dates/`ValidateNested` — isn't tagged yet;
their fields still show up via `classMetadata`, just with `decorator:
undefined`). `RTOFieldMetadata`/`ClassFieldDecoratorMeta` are the exported
types for this shape (`@zanix/utils/types`). Stacking two decorators on one
property (already discouraged — see `BaseRTO`'s own JSDoc) also makes the
`classMetadata` entry order-dependent (last decorator applied wins); don't
rely on which one "wins" if a property ever does carry more than one.

## Nesting and custom cross-field validation

```ts
@ValidateNested(AddressRTO)
accessor address!: AddressRTO
```

`ValidateNested(RTO, options?)` validates a nested `BaseRTO` (or, with
`each: true`, an array of them) by recursively running `classValidation`'s
internal logic — always self-exposes.

**Real footgun**: a nested accessor with a class-instance default (`=
new AddressRTO(...)`) has that default **bypass decorator validation
entirely** — since it was already constructed by code, not resolved from
the incoming payload, `classValidation` never re-validates it.

```ts
@Validation(function (this, value, property) {
  return value >= this.min // `this` is the merged (exposed properties + instance) data
}, { message: 'max must be >= min' })
accessor max!: number
```

`Validation(fn, options?)` is the generic custom-validation decorator for
anything the built-in catalog doesn't cover, including cross-field checks
— `fn(value, property)` runs with `this` bound to the merged exposed data,
can return `boolean | Promise<boolean>`. Building a reusable, app-specific
decorator (e.g. `IsObjectID`) on top of `Validation` is the idiomatic
pattern — don't reach for `defineValidationDecorator` for this.

## Ecosystem context

This is the most important module in the package — every other Zanix
package that deals with incoming data (routers, controllers, workers)
builds on top of it. `@zanix/server`'s route/handler config (`RtoTypes` —
`Body`/`Params`/`Search`) calls `classValidation` internally, so most
consumers never call it by hand.

## Checklist before authoring an RTO or a custom decorator

- [ ] Is every validated field declared `accessor`, never a plain class
      field?
- [ ] Does every field meant to appear on the validated instance carry
      `expose: true` (or self-expose automatically) — not silently dropped
      after passing validation?
- [ ] For a nested accessor with a class-instance default, is it understood
      that the default bypasses validation entirely — not re-checked on
      every `classValidation` call?
- [ ] Is a new app-specific rule built on `Validation()` rather than
      `defineValidationDecorator` unless it genuinely needs that lower
      level?
- [ ] Building something that needs a `BaseRTO`'s field shape without a real
      payload to validate (a generator, a renderer)? Reach for `classMetadata`
      instead of hand-rolling reflection or parsing the class.

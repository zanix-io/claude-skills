---
name: utils-interpolation-and-data-transforms
description: interpolate/interpolateEnv (the canonical home for the ${{ENV_VAR}}-missing-becomes-literal-'undefined' footgun referenced across the ecosystem), getPath, cleanRoute, processUrlParams, and the URL/query-string helpers (toSearchParams, getProcessedParams, interpolateUrl, sanitizeUrl). Use when template-interpolating a string, resolving env vars into config, or transforming route/URL data.
---

Covers `@zanix/utils/helpers`'s string-interpolation and URL/route
transformation utilities. File:line references point at
`~/Documents/Development/ZanixLibraries/utils` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- Check whether a string needs `interpolate` (`{{field}}`, real data) or
  `interpolateEnv` (`${{ENV_VAR}}`, environment) before reaching for
  either — they're deliberately non-colliding syntaxes, not
  interchangeable.
- `interpolateEnv`'s missing-var behavior (a literal `'undefined'` string,
  not an error) is the single most important fact in this skill — verify
  it before assuming a misconfigured deployment fails loudly.

## `interpolate`: real data into a template

```ts
import { interpolate } from 'jsr:@zanix/utils@[version]'

interpolate('Bearer {{token}}', { token: 'abc123' }) // 'Bearer abc123'
```

`interpolate<T>(value, record)` — an exact single `{{field}}` resolves to
the real typed value from `record` (not stringified); mixed text
(`'Bearer {{token}}'`) coerces to a string; recurses into arrays/objects.
Deliberately **skips** `${{...}}` syntax (leading `$`) so it never
collides with `interpolateEnv`'s own placeholder grammar — the two
conventions are meant to coexist in the same string, each resolving only
its own kind.

`getPath(record, path)` resolves a dot-path into `record`, including
array indices — the mechanism `interpolate` uses internally to look up
`{{a.b.0}}`-style paths. `matchWholePlaceholder(value)` checks whether a
string is *exactly* one placeholder (vs. embedded in surrounding text).

## `interpolateEnv`: the real, ecosystem-wide missing-var footgun

```ts
import { interpolateEnv } from 'jsr:@zanix/utils@[version]'

interpolateEnv('Bearer ${{API_KEY}}') // resolves against Deno.env
```

`interpolateEnv<T>(value)` resolves `${{ENV_VAR}}` placeholders against
`Deno.env`. **An unset env var substitutes as the literal string
`'undefined'` — never throws, never leaves the placeholder untouched.**
`'Bearer ${{MISSING}}'` with `MISSING` unset becomes the literal string
`'Bearer undefined'`, not an error and not `'Bearer ${{MISSING}}'`. This is
the canonical source of the `${{ENV_VAR}}`-missing footgun referenced
across the ecosystem (e.g. a trigger action's `url`/`headers` field, or any
other config resolved through this function) — a misconfigured deployment
degrades silently into a broken-but-syntactically-valid value rather than
failing at the point the missing var actually mattered. Always confirm a
required env var is genuinely set in the target deployment before trusting
a value that went through `interpolateEnv` — don't rely on this function to
surface the misconfiguration itself.

## Routes and URL params

```ts
cleanRoute('//api//users/') // '/api/users'
```

`cleanRoute(route, keepCase?)` — trims, converts backslashes to slashes,
collapses repeated slashes, forces a single leading slash, strips a
trailing slash, and lowercases unless `keepCase` is truthy.

```ts
processUrlParams({ name: 'Jos%C3%A9' }) // { name: 'José' }
```

`processUrlParams<T>(obj)` recursively `decodeURIComponent`s string values
in place; non-string values are untouched. **Real footgun**: if
`decodeURIComponent` throws partway through (a malformed `%` sequence),
the error is **swallowed** and the object is returned partially decoded up
to the point of failure — a silent partial-failure mode, not a thrown
error or an all-or-nothing guarantee.

## Query string / URL helpers

```ts
toSearchParams({ tags: ['a', 'b'], filter: { active: true } })
// tags=a&tags=b&filter[active]=true

getProcessedParams(new URLSearchParams('tags=a&tags=b&filter[active]=true'))
// { tags: ['a', 'b'], filter: { active: 'true' } }

interpolateUrl('https://x.com/{{id}}?tags={{tags}}', { id: '42', tags: ['a', 'b'] })
// 'https://x.com/42?tags=a&tags=b'
```

`toSearchParams(params)` — arrays become duplicate keys, nested objects
become bracket notation, `null`/`undefined` values are skipped entirely.
`getProcessedParams(searchParams)` is the reverse direction: simple keys →
value, repeated keys → array, `key[sub]=` → nested object.
`interpolateUrl(url, record)` interpolates the path portion as plain text
via `interpolate`, but expands a query segment that's exactly one
placeholder through `toSearchParams` instead — so an array value in
`record` correctly becomes repeated query keys rather than a stringified
array.

## `sanitizeUrl`: the navigable-`href`/`src` XSS guard

```ts
import { sanitizeUrl } from 'jsr:@zanix/utils@[version]'

sanitizeUrl('javascript:alert(1)') // ''
sanitizeUrl('data:image/png;base64,...') // unchanged (image data: is safe)
sanitizeUrl('data:text/html,<script>...') // ''
sanitizeUrl('/a/b?c=1') // unchanged
sanitizeUrl(undefined) // undefined (non-string values pass through)
```

`sanitizeUrl<T>(value)` neutralizes a value about to be used as a
navigable `href`/`src`: `javascript:`, `vbscript:`, and non-image `data:`
schemes are rejected (returned as `''`) instead of reaching the DOM.
Everything else — a safe URL string, or a non-string value — passes
through unchanged. Before the scheme check, ASCII tab/CR/LF are stripped
and leading C0-control/space is trimmed, matching the normalization a
browser applies when resolving a URL's scheme — without it,
`"java\tscript:alert(1)"` reads as harmless to a naive check but the
browser still executes it as `javascript:alert(1)`.

This is the one function every author-controlled URL-shaped prop (a
RichText/markdown tag, a template field, anywhere untrusted input reaches
a `href`/`src`) should be sanitized through — never re-implemented per
call site, so this bug class can only exist here. Real consumer:
`@zanix/space-ui`'s RichText tag renderer runs every `href`/`src` it emits
(links, images, iframes, video) through `sanitizeUrl` before handing it to
the underlying component.

## Checklist before interpolating or transforming a string/URL

- [ ] Is the right function chosen for the syntax actually present —
      `interpolate` for `{{field}}`, `interpolateEnv` for `${{ENV_VAR}}` —
      not assumed interchangeable?
- [ ] Is a value going through `interpolateEnv` verified to have its env
      var actually set in the target deployment — not trusted to fail
      loudly if missing?
- [ ] Does code consuming `processUrlParams`'s output handle a partially-
      decoded object as a real possibility, not assume all-or-nothing
      success?
- [ ] For a URL needing an array-valued query param, is `interpolateUrl`
      (which expands correctly) used instead of plain string interpolation
      (which would stringify the array)?
- [ ] Does a URL-shaped value from untrusted input (CMS, translation,
      user-controlled content) reach a navigable `href`/`src` only after
      going through `sanitizeUrl` — not re-implemented ad hoc per call
      site?

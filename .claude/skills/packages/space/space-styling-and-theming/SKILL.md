---
name: space-styling-and-theming
description: cssPlugin's typed CSS Modules/Tailwind default, responsive/scoped CSS delivery (global vs. per-page styles vs. automatic comet-scoped CSS, the media attribute's real effect, the global→page→comet cascade order), the design-token convention (primitive vs. semantic, --space-* naming, base-to-host precedence), and defineSpaceApp({ theme: { resolve } }) — the one real runtime, per-request token personalization API, including its sanitization/CSP/caching interactions. Use when writing component CSS, scoping a stylesheet to one page or component, declaring design tokens, or personalizing tokens per request (e.g. per-tenant branding).
---

Covers CSS delivery and design tokens. For the CSP mechanism `theme.resolve`
plugs into, see `space-middleware-and-security`; for `population`, the axis
`theme.resolve` most commonly branches on, see `space-i18n-and-population`.
File:line references point at `~/Documents/Development/ZanixLibraries/space`
— read the real code there before assuming this summary is still accurate.

## Golden rule (token savings)

- A component consumes semantic tokens (`var(--space-color-primary)`), never
  primitives directly — check this rule before writing new component CSS
  rather than re-deriving the primitive/semantic distinction each time.
- `theme.resolve` is the one genuine runtime API here; everything else in
  this skill is a static, build-time convention. Don't reach for a runtime
  mechanism to solve something the static token layer already covers.

## Build setup

```ts
// vite.config.ts
import { cometPlugin, cssPlugin, spacePlugin } from '@zanix/space/vite'
export default defineConfig({ plugins: [spacePlugin(), cometPlugin(), cssPlugin()] })
```

```ts
// main.ts — after activateApps(), before bootstrapServers()
import { loadCssManifest } from '@zanix/space'
await loadCssManifest('./dist/client/css-manifest.json')
```

`cssPlugin(options?)` default options: `{ tailwind: true, modules: true,
vanillaExtract: false }`. `modules: false` disables `*.module.css` → JS
class-map behavior app-wide (a `.module.css` then becomes a plain
side-effect CSS import). Every `*.module.css` gets a matching
`*.module.css.d.ts` next to it (dev and build alike), checked by `deno
check`/CI — a renamed/misspelled class name is a compile error, not a
runtime surprise. `loadCssManifest` reads the manifest back so a
full-document response links stylesheets automatically, hoisted into
`<head>` via React 19 resource hoisting regardless of whether the root
layout or the default shell owns `<head>`; an Orbit fragment never repeats
the **global** list (already present since the initial load) but does carry
the destination page's own page/comet stylesheets — see
`space-orbit-navigation`'s "CSS during navigation" for the dedupe/FOUC-
avoidance mechanism. In dev, `SpaceDevEngine` serves each declared
stylesheet directly (a `?direct` suffix on its URL, no manifest, no
hashing) — same shape, same `<link>`, just no build step in between.

## Responsive delivery: `media`, per-page `styles`, and comet-scoped CSS

Every stylesheet Space delivers — global, per-page, or a Comet's own
`*.module.css` — is a `StylesheetRef`: a plain path/URL string, or
`{ href, media }` for one that should carry a `media` attribute through to
the rendered `<link>`.

- **Global** (`defineSpaceApp({ globalCss: [...] })`) — order matters, a
  later entry can override an earlier one, preserved in `css-manifest.json`
  regardless of the hashed output filenames.
- **Per page** — a `static styles: StylesheetRef[]` field on the page
  controller, resolved relative to that page's own file (co-located, like a
  Comet's own `import './x.module.css'`, deliberately not root-relative
  like `globalCss`). Genuinely scoped: linked only on a response for that
  page. Not yet composed with a layout's own `styles` — only a page's own
  direct declaration resolves today.
- **Per component** — a Comet's own `*.module.css` import is scoped
  automatically, no field to declare; `cssPlugin` correlates each Comet's
  build entry to the CSS it actually imports. **This fixed a real bug**:
  before this correlation existed, `generateBundle` swept every built
  `.css` asset into one flat global list, so a Comet used on one page out
  of fifty shipped its CSS on all fifty.

All three render under the same cascade, in the same order — **global →
page → comet** — ordinary CSS specificity applies on top (a heavier
selector earlier still wins).

**What `media` does and doesn't do**: a non-matching `media` query still
downloads (the browser needs its CSSOM ready in case a resize/rotation
makes it match later) but doesn't block rendering and fetches at lower
priority. **`media` avoids render-blocking — it does not reduce bytes
transferred or requests made.** The bytes/requests reduction comes from
scope (comet/page CSS not shipping where it's unused), a separate,
orthogonal mechanism. No breakpoint preset ships — write the `media` query
explicitly, or mirror whatever scale the app's CSS framework already uses.

**Manifest shape** (`css-manifest.json`, read via `getCssManifest()`):
`{ global: StylesheetRef[], pages?: Record<path, StylesheetRef[]>, comets?:
Record<sourceUrl, StylesheetRef[]> }`.

## Design tokens

No seeded palette ships by default — "a disconnected, unused default is
worse than none." Declared as plain CSS custom properties, imported once
from the root layout via `defineSpaceApp({ globalCss })`.

- **Primitive tokens**: raw values, no meaning — `--space-blue-500: #2563eb`.
- **Semantic tokens**: name a role, reference a primitive —
  `--space-color-primary: var(--space-blue-500)`.
- **A component must only ever consume semantic tokens**, never primitives
  directly.

**Naming**: prefix every token `--space-*`. Never collide with a
third-party's own internal prefix — the explicit real example: never
`--tw-color-primary`, since `--tw-*` is Tailwind's internal namespace, and a
collision "silently breaks Tailwind's own utilities in ways that are hard to
trace back." Semantic names read as a role, not an implementation
(`--space-color-primary`, never `--space-blue-main`). If the app itself is a
base for others, semantic token names are a public contract — renaming one
is a breaking change.

**What a component should do**: reference via `var(--space-color-primary)`;
fall back sensibly (`var(--space-color-primary, #2563eb)`) only if it
genuinely needs to work outside this app's own token sheet; treat the token
NAME as the contract, never its current value.

**What a component should not do**: hardcode a literal that maps to an
existing semantic token; reference a primitive directly from component CSS;
invent a component-local custom property duplicating an existing semantic
token (`--btn-primary-color` instead of reusing `--space-color-primary`);
assume a token is only ever one literal value across the app's lifetime.

**Light/dark**: no Zanix-specific mechanism — plain CSS, the standard
`@media (prefers-color-scheme: dark)` + `:root:not([data-theme='light'])` +
`:root[data-theme='dark']` override pattern. `[data-theme]` is an ordinary
attribute the app's own root-layout/client code sets; the package has no
opinion on how the preference is stored.

**Base→host precedence**: rides `globalCss`'s own composition
(`addGlobalCssPaths`) — the base app's `globalCss` executes first, the
host's own executes after; `getGlobalCssPaths()` resolves to e.g.
`['./tokens.css', './host-tokens.css']`, and normal CSS cascade means the
host's later `:root` rule wins for redeclared properties. A host never needs
to know the base app's own primitive scale.

## Runtime personalization: `defineSpaceApp({ theme: { resolve } })`

The one genuine runtime API in theming — every other field on `SpaceAppConfig`
for theming doesn't exist; theming is otherwise a convention, not an API.

```ts
export default defineSpaceApp({
  name: 'storefront',
  theme: {
    resolve: ({ population }) =>
      population === 'tenant-b' ? { '--space-color-primary': '#16a34a' } : undefined,
  },
})
```

`resolve(ctx)` receives `{ population, lang, request }` — `population` is
the same segment/tenant id `populationGuard`/`ctx.population` resolve
(`space-i18n-and-population`); `lang` comes from the `:lang` route param
when the app follows the `routes/[lang]/...` convention, else `undefined`;
`request` is the raw `Request`. Returns `Record<string, string> |
undefined` (or `{}`) — a map of `--space-*` overrides; `undefined`/`{}`
means no override for that request.

Injected as a small, nonced `<style>` block on every full-document response,
positioned to override the static stylesheet's `:root` via normal cascade.
**App-wide only** — no per-page override in this version.

**Sanitization**: a token name must be a real custom-property name
(`--foo-bar`); a value containing `;`/`{`/`}`/`<`/`>`/backtick/newline is
dropped entirely, never interpolated.

**CSP**: the injected `<style>` needs `style-src` to permit its nonce —
`SpacePageController`'s zero-config default CSP already grants this
unconditionally, even for apps not using `theme` at all
(`space-middleware-and-security`). A page/app that supplies its **own**
custom CSP (replacing the default) must grant its own `style-src` + matching
nonce — the exact same disclosure already required of a custom policy that
restricts `script-src` against the inline initial-state script.

**Caching**: a page combining `cacheControl` with a configured
`theme.resolve` automatically folds `population` into its `ETag` — closing
the gap where two populations sharing identical loader data could otherwise
collide on `ETag` and serve a stale `304` with the wrong branding. This is
**deliberately narrow**: it does not make `@zanix/space` population-aware
for caching in general, and says nothing about a shared/CDN cache's own
partitioning (see `space-i18n-and-population`'s caution — nothing in
`@zanix/space` assumes a shared cache exists). `cacheControl` itself remains
the page author's explicit responsibility.

## Styling the framework's own markup

CSS attribute selectors, no override prop needed: `[data-space-outlet]`
(the Orbit outlet, `space-orbit-navigation`), `[data-comet]`/
`[data-comet-strategy="visible"]` (Comet boundaries, `space-comets`). Both
default to `display: contents` inline, so they never break a parent
grid/flex by inserting an extra box — override with more specific CSS only
if a real box is actually needed.

## Fonts and critical resources

Use `react-dom`'s own `preload`/`preinit`/`preconnect` directly, no
framework API needed:

```tsx
import { preload } from 'react-dom'
function RootLayout({ children }) {
  preload('/fonts/inter-var.woff2', { as: 'font', type: 'font/woff2', crossOrigin: 'anonymous' })
  return <html>...</html>
}
```

## Checklist before adding component CSS or a design token

- [ ] Does new component CSS reference only semantic tokens, never a
      primitive directly?
- [ ] Does a new token's name avoid colliding with a third-party's own
      prefix (`--tw-*` and equivalents)?
- [ ] If personalizing per-request, is `theme.resolve` the actual mechanism
      — not a component-level workaround for something that's really a
      request-time concern?
- [ ] If a page uses both `cacheControl` and `theme.resolve`, is the
      automatic `population`-in-`ETag` folding actually sufficient — or does
      this app also need its own shared-cache partitioning?
- [ ] Is a stylesheet declared at the narrowest scope that actually needs
      it — global only for truly app-wide CSS, `static styles` for one
      page, a plain Comet-local `*.module.css` import for one component —
      rather than defaulting to `globalCss` out of convenience?
- [ ] If a stylesheet uses `media`, is it understood as render-blocking
      avoidance, not a way to cut bytes/requests — scope is what does that?

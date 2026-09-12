---
name: space-assets-and-media
description: Static asset serving (defineSpaceApp assetsDir), content-hashed assets (assetsPlugin/resolveAssetHref), build-time-only image/SVG optimization with the never-worsen rule and the SVG-sprite-id preservation footgun, media transformation (video/thumbnail/voice via real ffmpeg), and the canonical explanation of defineSpaceApp({ clientBuildDir }) — the mechanism that auto-loads all seven production manifest/build-output loaders (comets, client entry, CSS, assets x2, PWA, sitemap) from a production main.ts, referenced by space-pwa for its own slice. Use when serving a static file, optimizing an image/SVG, transcoding video/audio, or wiring a real build's output at runtime.
---

Covers everything under `defineSpaceApp({ assetsDir })` and the build-time
optimization plugins layered on top. For PWA icon generation (a separate
pipeline), see `space-pwa`. File:line references point at
`~/Documents/Development/ZanixLibraries/space` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- The never-worsen rule (pure byte-size comparison) governs every
  optimization variant this skill covers — trust it rather than
  second-guessing whether a specific format/breakpoint combination is safe.
- Reference an asset by its stable path (`/assets/logo.svg`), not by
  `import` — check this before assuming an asset needs a code change to be
  overridable.
- `defineSpaceApp({ clientBuildDir })` already wires the hashed-assets
  manifest (and six other production loaders) into a real `main.ts`
  automatically — don't add a manual `loadAssetsManifest`/
  `loadAssetsBuildOutput` call unless the app genuinely doesn't set
  `clientBuildDir`.

## Static asset serving

```ts
export default defineSpaceApp({ name: 'storefront', assetsDir: './assets' })
```

```tsx
<img src='/assets/logo.svg' alt='Logo' />
```

`assetsDir` accepts a string or `string[]` — array form composes
first-match-wins by relative path (`['./assets-override',
'./node_modules/@acme/shop-app/assets']`), resolved into one `Map<relativePath,
absolutePath>`. Omitted by default (opt-in, unlike `routesDir`). Served over
`@zanix/server`'s trailing catch-all route, looked up directly in the Map —
no filesystem concatenation, unmapped path → 404 — the exact same code path
in `znx space dev` and production. Case-sensitive like a real filesystem
(`/assets/Logo.svg` ≠ `/assets/logo.svg`).

**Real footgun, worth stating explicitly**: an asset is only overridable if
referenced by this stable public path — never via a bare `import logo from
'./logo.svg'` inside a component. A bare import resolves through Vite's
module graph, independent of `assetsDir`; module-aliasing for the import
case isn't built.

## Content-hashed assets: `assetsPlugin`/`resolveAssetHref`

```ts
// vite.config.ts
import { assetsPlugin, spacePlugin } from '@zanix/space/vite'
export default defineConfig({ plugins: [...spacePlugin(), assetsPlugin({ assetsDir: './assets' })] })
```

```ts
// space.app.ts — `clientBuildDir` makes `setup()` load `assets-manifest.json`
// (and the rest of a real build's output) automatically; see "Wiring the
// client build output at runtime" below
import { defineSpaceApp } from '@zanix/space'
export default defineSpaceApp({ name: 'storefront', clientBuildDir: './dist/client' })
```

```tsx
import { resolveAssetHref } from '@zanix/space'
<img src={resolveAssetHref('logo.svg')} alt='Logo' />
```

Writes `assets-manifest.json` during a real `zanix space build`.
`resolveAssetHref(path)` returns the hashed URL (`/assets/logo-a1b2c3.svg`)
if the manifest is loaded, else falls back to the stable unhashed path —
never throws, never asserts existence. Serving does two lookups in order:
hashed build output first (`Cache-Control: public, max-age=31536000,
immutable` + a real `ETag`, the hash IS the filename), then falls back to
the unhashed lookup with no special caching.

## Wiring the client build output at runtime

`defineSpaceApp({ clientBuildDir })` is the primary, documented way a
production `main.ts` wires up a real build's output — set once (e.g.
`'./dist/client'`, matching `zanix space build`'s own default `--out-dir`),
its own `setup()` (which runs when `activateApps()` runs) automatically
calls all seven manifest/build-output loaders this same directory feeds, in
the required order: `loadCometManifest`, `loadClientEntryManifest`,
`loadClientEntryProductionKey`, `loadCssManifest`, `loadAssetsManifest`,
`loadAssetsBuildOutput`, `loadPwaBuildOutput`, then `loadSitemapManifest`
(this last one only matters for `sitemap: 'auto'`). A production `main.ts`
no longer has to call any of these seven itself.

Omitted entirely by default — no file is read, at zero cost, same
convention as `assetsDir`/`pwa`/`sitemap`. Skipped entirely under
`znx space dev` (`isDevClientEnabled()`) — not because a missing manifest
file would be an error (each loader already tolerates that fine), but
because a STALE one on disk very much isn't: a real `zanix space build` and
`znx space dev` commonly point at the same `clientBuildDir` on the same
machine, and an earlier build's real output already sitting there is the
common case in local development, not a rare one. Loading it under dev
would resolve every Comet/stylesheet/asset to that OLD build's own hashed
names instead of the current source Vite is actually serving.

Calling one of the seven loaders directly in `main.ts` is now only for an
app that doesn't set `clientBuildDir` at all — an unusual layout the option
doesn't fit (e.g. a build output split across more than one directory).
Setting `clientBuildDir` AND calling one of the seven by hand for the same
app double-loads that one manifest harmlessly (the second call simply
overwrites the first with the same file's own contents).

## Image/SVG optimization: `assetsPlugin({ optimize })`

Build-time only, using real `sharp`/`svgo` — never run in the deployed
server. **Universal rule**: an optimized output only replaces/adds a variant
if strictly smaller in bytes, measured, never assumed. Equal-or-larger keeps
the original bytes exactly.

```ts
assetsPlugin({
  assetsDir: './assets',
  optimize: {
    images: { breakpoints: ['msm', 'mlg', 'dlg'], formats: ['webp'] },
    svg: true,
    include: ['img/**'],
  },
})
```

- `images: true` (bare) — in-place recompress of the original key's bytes
  only, metadata stripped by sharp's default (no `.withMetadata()`).
- `images: { breakpoints }` — additive, original key untouched. Named
  presets `thum`/`msm`/`mlg`/`dmd`/`dlg` (overridable via `quality`/`width`),
  or a raw pixel width (`720` → key `w720`). `withoutEnlargement: true`.
  Each variant is compared against the global original; emitted (e.g.
  `logo.msm.jpg`) only if smaller.
- `images: { formats }` (no breakpoints) — each format encoded at original
  dimensions, each compared independently against the global original
  (never against each other).
- `images: { breakpoints, formats }` — a three-tier comparison: each
  breakpoint's own same-format resize is the reference its own formats
  compare against (e.g. `logo.msm.webp` must beat `logo.msm.jpg`, not the
  bare `logo.jpg`).
- `svg: true` — runs `svgo` (works under Deno, no native binary), safe
  transforms only (strip dimensions/metadata/comments, minify inline
  styles/ids), deliberately not a whole-app CSS-selector purge.

**Real SVG-sprite footgun, mitigated automatically but worth knowing**:
`<symbol id="...">` sprite ids are auto-protected from `cleanupIds` on every
file, always — no config needed (verified against `@zanix/space-ui`'s own
17-symbol `catalog.svg`: bare `svg: true` keeps all 17). The mechanism
scans each file's raw source for `<symbol id="...">` before svgo runs and
passes the list to svgo's `cleanupIds` `preserve` option. Non-symbol dead
ids in the same file are still cleaned normally.

For a **non-symbol** id referenced externally (e.g. `clip-path:
url(other-file.svg#id)`), the auto-protection doesn't apply — use
`svg: { preserveIds: ['icons/**'] }` instead:

```ts
assetsPlugin({ assetsDir: './assets', optimize: { svg: { preserveIds: ['icons/**'] } } })
```

A matching file skips `cleanupIds` entirely — **`remove: false` alone is not
enough**, since `minify: true` still rewrites surviving id text even when
removal is disabled. `preserveIds` is the actual escape hatch, not a partial
svgo config.

Other options: `include` (glob patterns against manifest relative-path
keys, omitted = every eligible asset considered); `useWorker` (offloads
sharp/svgo work to `@zanix/utils`'s `WorkerManager` — `true` sizes the pool
to detected CPU count, a `number` sets it explicitly; purely an execution
strategy, same emit/discard decisions either way). Every variant is just
another `assets-manifest.json` entry, resolved via `resolveAssetHref`.

## Media transformation (video/thumbnail/audio)

```text
Asset Transformation API (createAssetTransformer)
├── image      — sharp, via assetsPlugin({ optimize: { images, svg } })
├── video      — real system ffmpeg, via mediaPlugin({ optimize: { video } })
├── thumbnail  — real system ffmpeg, via mediaPlugin({ optimize: { thumbnails } })
└── audio      — real system ffmpeg, via mediaPlugin({ optimize: { audio } })
     └── voice — the only implemented audio PROFILE today; music/podcast/... are extension points, not implemented
```

`mediaPlugin({ optimize: { audio: { voice: {...} } } })` covers voice/speech
optimization only, not generic audio. Formats: `aac` (`.m4a`,
universal-compatibility fallback) and `opus` (`.opus`) — the two encoders
ffmpeg provisioning already guarantees; MP3/Vorbis/FLAC are explicitly
excluded. Bitrate: a single fixed target (`128kbps` default, overridable via
`bitrateKbps`) — no breakpoints/CRF/CQ/`maxrate`/`bufsize`. Only `.wav`
sources are ever transcoded — already-compressed lossy files (`.mp3`/`.m4a`/
`.opus`) are left untouched even with `audio.voice` configured, since
re-encoding lossy risks quality loss for uncertain savings. Same
never-worsen (byte-size) rule as images. An unsupported profile/format is a
real, actionable error, never a silent fallback.

`@zanix/space` never installs/downloads ffmpeg itself — see `@zanix/cli`'s
`deploy.md` for Docker provisioning (the same `aac`/`libopus` encoders
video/thumbnails already require).

## Checklist before touching asset/media configuration

- [ ] Is a new asset referenced by its stable `/assets/...` path — never a
      bare `import`, if it needs to be overridable/host-composable?
- [ ] For a new SVG with externally-referenced non-symbol ids, is
      `preserveIds` used — not just `remove: false`, which `minify: true`
      would still undo?
- [ ] Is a breakpoint/format optimization actually compared against the
      right baseline (own-format breakpoint vs. bare original) for the
      claim being made about savings?
- [ ] For audio, is the source genuinely `.wav` — lossy sources are silently
      left untouched even with `audio.voice` configured, by design?
- [ ] Does `clientBuildDir` (or, for an app not using it, `main.ts`'s own
      manual loader calls) genuinely match the client build's real output
      directory — checked directly, not assumed consistent?

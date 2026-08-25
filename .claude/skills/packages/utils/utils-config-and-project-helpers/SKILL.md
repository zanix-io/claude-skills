---
name: utils-config-and-project-helpers
description: Config file read/save (with its in-memory cache, and resetConfig to clear it in a test), path/temp-folder helpers, the global Zanix/Znx namespace accessors (including its lazy config getter), mockWrap for source-rewriting test doubles, and which project-scaffolding types/functions moved to @zanix/cli and no longer live here. Use when reading/writing deno.json(c) programmatically, resolving a path relative to a module, or mocking a dependency in a test.
---

Covers `@zanix/utils/helpers`'s config/path/Zanix-namespace utilities and
`mockWrap` (from the `/testing` subpath). File:line references point at
`~/Documents/Development/ZanixLibraries/utils` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- Check the "moved to @zanix/cli" list before assuming a project-scaffolding
  helper still lives in this package — several real functions/types were
  relocated, and old docs/examples referencing them are stale.
- Use `saveConfig` (which resets `readConfig`'s cache) rather than writing
  the config file by hand when a read-modify-write cycle needs to see its
  own change. In a test that needs `readConfig()` to re-read a fresh file
  from disk instead, use `resetConfig()` (`jsr:@zanix/utils/testing`)
  directly — test-only, production code relies on the cache staying warm.

## Config file access

```ts
import { getConfigDir, getRootDir, readConfig, saveConfig } from 'jsr:@zanix/utils@[version]/helpers'

const root = getRootDir() // Deno.cwd()
const configDir = getConfigDir(root) // resolves deno.json/deno.jsonc, prefers .json; null if neither exists
const config = readConfig(configDir)
config.version = '1.2.3'
await saveConfig(config, configDir)
```

**`readConfig` caches in memory, keyed by `configPath`** — a stale read is
possible if the file changes on disk without going through `saveConfig`
(which resets the cache). `readConfig` **throws** if no config file can be
resolved at all — no silent `undefined`. `readModuleConfig(metaUrl,
isJsonc?)` reads a local `file:` config, or fetches the JSR equivalent when
running from a published module — `isJsonc` default `true`.

`fileExists(path)`/`folderExists(path)` require `allow-read`, but **swallow
every `Deno.statSync` failure into a silent `false`** — a bare `catch`
with no error-type filter, so a permission denial (`Deno.errors.NotCapable`/
`PermissionDenied`) is indistinguishable from a not-found path. A missing
`allow-read` grant looks identical to "file doesn't exist" — a silent-false
footgun, not a loud one.

## Paths and temp folders

```ts
getRelativePath(to, from?) // from defaults to getRootDir()
getPathFromCurrent(callerUrl, relativePath)
getTemporaryFolder(callerUrl, unique?) // creates/returns a __tmp__ folder next to callerUrl
```

**Real footgun**: without `unique`, repeated `getTemporaryFolder` calls
return the *same* folder — not isolated across concurrent callers. Pass
`unique` (a string sets its prefix) for concurrency-safe scratch space.
Intended as git-ignored scratch space — confirm `.gitignore` actually
covers `__tmp__` in a project using this.

### `confinePath`: path-traversal guard

```ts
confinePath(rootDir, key) // resolves key against rootDir, throws if it escapes
```

The guard any storage/filesystem layer needs before touching disk with a
caller-supplied `key` (or an id used to build one): resolves `key` against
`rootDir` and throws `ApplicationError` (`UTILS_PATHS_TRAVERSAL_BLOCKED`) if
the result lands outside `rootDir` — catching both a `../`-segment escape
and an absolute `key` that overrides `rootDir` entirely, with the same one
containment check. `key` resolving to `rootDir` itself (an empty/`.` key)
is also rejected — a storage `key` always names something *inside* the
root, never the root.

## The global `Zanix`/`Znx` namespace

```ts
import { canUseZnx, getGlobalZnx, setGlobalZnx } from 'jsr:@zanix/utils@[version]/helpers'

if (canUseZnx()) {
  const znx = getGlobalZnx()
}
```

`setGlobalZnx(data)` initializes `Znx` on first call and merges `data` into
it on every call — but `config` itself resolves **lazily**: reading
`deno.json`'s own `zanix` field (via `readConfig()`, ignoring read errors)
is deferred to the first time something actually accesses `Znx.config`, not
to the `setGlobalZnx` call itself. `Znx.config` self-materializes into a
plain, writable property after that first real read (still directly
mutable afterward, e.g. `Znx.config.project = 'space'`). This is the same
global accessor `utils-logger`'s `Znx.logger`/`self.logger` assignment
writes into. `resetConfig()` (`jsr:@zanix/utils/testing`) clears
`readConfig()`'s own module-level memoized result for a test that needs to
control what a *fresh* `Znx.config` resolves to — it does NOT
un-materialize an already-resolved `Znx.config` on an existing `Znx` object.

**`Znx` gets seeded as an import-time side effect, not only via an explicit
call — but that no longer touches disk.** `@zanix/utils/logger`'s own
module body runs `new Logger()` unconditionally on import
(`modules/logger/mod.ts`), and `Logger`'s constructor calls
`setGlobalZnx(globals)` unconditionally too — outside the
`disableGlobalAssign` check, which only gates whether `globals.logger`
itself gets assigned (`modules/logger/main.ts`). So `canUseZnx()` returns
`true` the moment ANY code path imports `@zanix/utils/logger` (directly or
transitively), before any user code runs `setGlobalZnx`/`new Logger()`
itself — `canUseZnx()` only returns `false` in a process that hasn't
imported that module at all. Thanks to the lazy `config` getter above,
though, this import-time seeding no longer performs a synchronous
`readConfig()` disk read on its own — only actually reading `Znx.config`
does (e.g. `baseSaveData`'s `Znx.config.project` check, evaluated per save
call, not at `Logger` construction). This is exactly the test-isolation
footgun behind `@zanix/utils`'s own `namespace.test.ts`/`logger.test.ts` —
order matters, and importing the logger module anywhere in a test run
seeds the global for every test after it.

## Moved to `@zanix/cli` — no longer in this package

`getZanixPaths`, `getAllZanixLibrariesInfo`, `getLatestVersion`/
`getLatestRelease`, `ZanixTree` (and its `getServerSrcTree`/
`getSpaceSrcTree`/`getLibrarySrcTree` builders), `prepareGithub`,
`createVSCodeConfig` — all project-tree scaffolding and GitHub/editor
bootstrapping runtime implementations. Anything referencing these from
`@zanix/utils` is stale; see `@zanix/cli`'s own `engineering.md` §5/§7 for
where they live now. The `Zanix*SrcTree`/`ZanixFolderTree`/`ZanixLibraries`
**types** (no runtime implementation) still live in `@zanix/utils/types`.
Repository-bootstrapping option types
(`BaseEditorHelperOptions`/`BaseGithubHelperOptions`/`Editors`/
`HookOptions`/`PreCommitHookOptions`/`PrepareGithubOptions`/
`WorkflowOptions`) and `CompilerOptions` (tied to a removed
`compileAndObfuscate`) moved too — referencing them from
`@zanix/utils/types` also fails.

## `generateUUID`

```ts
generateUUID() // crypto.randomUUID()
```

## `isPlainObject`

```ts
isPlainObject(value: unknown): value is Record<string, unknown>
```

A type guard for "a genuine plain object" — rejects `null`, arrays, and
class instances (checks `Object.getPrototypeOf(value) === Object.prototype`,
not just `typeof value === 'object'`). Real consumers outside this package:
`datamaster`'s DLQ filter (`modules/dlq/filter.ts`) and `space-ui`'s
RichText markdown/tags/prop-diffing (`components/RichText/{markdown,tags,
props-sentinel}.ts`) — anywhere a value's shape needs confirming before
spreading/merging it as an object.

## `mockWrap`: source-rewriting test doubles

```ts
import { mockWrap } from 'jsr:@zanix/utils@[version]/testing'

const context = {
  getRootDir: () => '/mock/root/dir/',
  join,
  fileExists: (filePath: string) => filePath === '/mock/root/dir/config.json',
  CONFIG_FILE: 'config.json',
}
const mockedGetConfigDir = mockWrap(getConfigDir, context)
mockedGetConfigDir() // "/mock/root/dir/config.json"
```

`mockWrap(fn, context, force?)` rewrites `fn`'s source so identifiers
matching `context`'s own keys rebind to `this.<key>`, returning a function
bound to `context` — a way to substitute a function's real dependencies
(other functions, constants) with mocks, without a DI container.

**Real footgun**: when a `context` value is a function and `force` isn't
set, only **call sites** (`key(`) are rewritten — a plain reference to that
same identifier elsewhere in `fn`'s body is left untouched. Pass `force:
true` to rewrite every occurrence unconditionally if a mock needs to
apply everywhere the identifier appears, not just where it's called.

## Checklist before using a config/path/mock helper

- [ ] Is `saveConfig` used for a read-modify-write cycle, so `readConfig`'s
      cache doesn't serve a stale value on the next read?
- [ ] Is a `false` from `fileExists`/`folderExists` being read as "doesn't
      exist" without ruling out a swallowed permission denial (missing
      `allow-read`) — the two are indistinguishable from the return value
      alone?
- [ ] Is a caller-supplied `key`/id resolved against a root directory before
      hitting disk — does it need `confinePath` to reject a `../` or
      absolute-path escape?
- [ ] Does `getTemporaryFolder` need `unique` — is more than one concurrent
      caller sharing the same `callerUrl` scratch space a real risk here?
- [ ] Before reaching for a project-scaffolding helper from this package,
      has the "moved to `@zanix/cli`" list been checked — it might no
      longer live here?
- [ ] For `mockWrap`, does the mock need `force: true` — are there plain
      (non-call-site) references to the mocked identifier in the wrapped
      function that also need substituting?

---
name: utils-linter-plugins
description: The four Deno.lint.Plugin exports (deno-fmt-plugin, deno-std-plugin, deno-test-plugin, deno-zanix-plugin) and their rules — including that deno-zanix-plugin silently bundles every rule from the other three, and the exact deno-std-plugin subpath spelling. Use when configuring deno.json(c)'s lint.plugins, or reviewing a lint finding from one of these plugins.
---

Covers `@zanix/utils/linter/*`'s four lint plugins. File:line references
point at `~/Documents/Development/ZanixLibraries/utils` — read the real
code there before assuming this summary is still accurate.

## Golden rule (token savings)

- Enabling `deno-zanix-plugin` alone already includes every rule from the
  other three — don't also list them individually unless the intent is
  genuinely to enable only a subset.
- Double-check the exact subpath spelling before wiring a new plugin —
  `deno-std-plugin`, not `deno-standard-plugin`.

## The four plugins

```jsonc
// deno.json(c)
{ "lint": { "plugins": [
  "jsr:@zanix/utils/linter/deno-fmt-plugin",
  "jsr:@zanix/utils/linter/deno-std-plugin",
  "jsr:@zanix/utils/linter/deno-test-plugin",
  "jsr:@zanix/utils/linter/deno-zanix-plugin"
] } }
```

| Plugin | Subpath | Rules |
| --- | --- | --- |
| Format | `linter/deno-fmt-plugin` | `single-quote`; `line-width` (max 100 chars, exceptions for `import` lines/comments/etc.) |
| Standard | `linter/deno-std-plugin` | `no-require`; `no-useless-expression`; `require-access-modifier` (constructors and `#private` members exempt) |
| Test | `linter/deno-test-plugin` | `no-only` (`Deno.test.only`); `no-ignore` (`Deno.test.ignore`) |
| Zanix | `linter/deno-zanix-plugin` | `no-znx-console`; `no-explicit-znx-imports`; `use-znx-flags`; **plus every rule from the other three, combined** |

**Real gotcha — `deno-zanix-plugin` silently bundles the other three's
rules.** Its own `mod.ts` spreads all of them into its own `rules` object
— enabling `deno-zanix-plugin` alone also opts into `single-quote`/
`line-width`/`no-require`/etc., even without listing the other plugins.
Diagnostics from those combined rules still report their `id` prefixed
with `deno-zanix-plugin`, not the original sub-plugin's name — don't
assume an `id` starting with `deno-zanix-plugin/` means the rule is
Zanix-specific.

Programmatic use (e.g. testing a rule in isolation):

```ts
import formatPlugin from 'jsr:@zanix/utils@[version]/linter/deno-fmt-plugin'

Deno.lint.runPlugin(formatPlugin, 'test.ts', `const a = "This is double quoted";`)
// [{ id: 'deno-fmt-plugin/single-quote', message: '❌ Use single quotes instead of double quotes.', ... }]
```

## `use-znx-flags`

```ts
Deno.lint.runPlugin(zanixPlugin, 'test.ts', `'otherFlag'`)
// [{ id: 'deno-zanix-plugin/use-znx-flags', message: '❌ The flag "otherFlag" is invalid.', hint: 'Review available flags:\n use comet', ... }]
```

Validates the directive-prologue string-literal grammar slot (same
position as `'use strict'`) against `ZNX_FLAGS` (`jsr:@zanix/utils/constants`
— currently `['use comet']`, the directive `space-comets` documents). **Only
checks the very first bare string-literal statement in a file** — a string
literal appearing anywhere else is ignored, not flagged.

## Authoring a new rule

Every real rule in `linter/plugins/zanix/rules/*.ts` follows the identical
shape — confirmed across all three (`zanix-flags.ts`, `zanix-imports.ts`,
`zanix-logger.ts`), not just one example generalized:

```ts
const rules: Record<string, Deno.lint.Rule> = {
  'rule-name': {
    create(context) {
      return {
        '<AST selector>'(node) {
          context.report({ node, message: '...', hint: '...' })
        },
      }
    },
  },
}
export default rules
```

One rule name per file is the established convention, not a hard
requirement — match the granularity of the sibling closest to the new
rule's own concern. Add the new file's rules to `linter/plugins/zanix/mod.ts`'s
combined export the same way the other three already are.

## Adding auto-fix to a rule

`Deno.lint`'s `ReportData.fix?(fixer: Fixer): Fix | Iterable<Fix>` is real
and supported — confirmed via `deno lint --fix` against a real target file,
not assumed from the type signature alone. `no-znx-console`
(`linter/plugins/zanix/rules/zanix-logger.ts`) already ships one, and is
the established precedent to follow — not a hypothetical. It rewrites
`console.log`/`.info`/`.warn`/`.error` to `logger.debug`/`.info`/`.warn`/
`.error` (any other `console.*` method has no safe 1:1 mapping and is
reported without a `fix`), and it already solves the three real
constraints below:

- **A rule's `create(context)` has no access to the target project's
  `deno.json(c)` imports map** — `RuleContext` only exposes `filename`,
  `sourceCode`, `report()`. `no-znx-console`'s `findLoggerAlias(filename)`
  reads the config file directly off disk, walking up from `ctx.filename`
  (bounded by the filesystem root, nearest config wins, never merges with
  a further ancestor) — a real, if unusual, use of file I/O inside a lint
  rule; no other rule in this plugin does this. **Never hardcode one
  alias.** This ecosystem has at least two live conventions for the same
  target (`@zanix/logger` in most repos, `@zanix/utils/logger` in
  `space-ui`) — `isLoggerSubpathTarget` matches either shape, and the
  actual alias string always comes from the target project's own config.
- **If the target project doesn't have the dependency the fix needs at
  all, don't offer the fix** — `findLoggerAlias` returns `null` when no
  resolvable alias exists; `no-znx-console` still calls `context.report()`
  (the rule's existing value) but leaves `reportData.fix` unset, so
  `--fix` leaves the file untouched rather than inserting an import that
  can't resolve.
- **A file with multiple violations of the same rule must not get the
  same import inserted once per violation.** `context.report()` is called
  once per matched AST node — each call's own `fix()` is independent
  unless something coordinates them. `no-znx-console` resolves the alias
  lazily on the first fixable violation and caches the result (including
  the `null` case) in a `resolution` closure variable scoped to
  `create(context)`'s own call — one resolution per file. It also reuses
  an existing `logger` import already bound in the file
  (`findExistingImportLocal`) instead of inserting a second one, and only
  the first fix in a file that actually needs a new import queues the
  `insertTextBeforeRange` edit — every later violation in the same file
  only rewrites its own call site via `replaceTextRange`.

**Two real setup gotchas when testing a fix in isolation, confirmed on
first real attempt:**

- **`Deno.lint.runPlugin` only works inside `deno test`** — it throws
  `"only available in deno test subcommand"` under plain `deno run`. Wrap
  any isolated plugin-testing script in a `Deno.test(...)` block, or use
  `deno lint --fix` against a real target file instead (which doesn't have
  this restriction).
- **Testing a local plugin file against an external scratch project needs
  the scratch project's own `deno.json` to carry this package's bare
  specifiers** (`modules/`, `utils/`, `@std/path`, etc.) — Deno resolves
  bare specifiers against one process-wide import map, not per-nearest-
  config, so a scratch project that only imports the plugin file by local
  path still needs those entries copied into its own `imports` map or the
  plugin's own internal imports won't resolve.

## Checklist before configuring or extending lint plugins

- [ ] Is the subpath spelled `deno-std-plugin`, not `deno-standard-plugin`?
- [ ] If `deno-zanix-plugin` is enabled, is it understood to already
      include the other three's rules — not layered on top of them
      redundantly, and not assumed to be Zanix-only in scope?
- [ ] Does a new directive-prologue-style flag actually need checking at
      the very first statement position — `use-znx-flags` won't catch it
      anywhere else in the file?
- [ ] (Auto-fix only) Does the fix resolve the target project's real
      import alias instead of hardcoding one, skip entirely when the
      needed dependency isn't available, and insert an import at most
      once per file even with multiple violations?

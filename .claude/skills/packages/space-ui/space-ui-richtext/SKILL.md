---
name: space-ui-richtext
description: @zanix/space-ui's RichText component — ICU rich-text tags via formatRichText, population via a typed sentinel (never a string round-trip), Markdown through a pure-AST parser with zero preact/compat, and the resolveRichTextDocument loader pattern. Use when rendering rich content, adding/changing a RichText tag, or reviewing a change that touches ICU/Markdown parsing.
---

For the "presents data, never owns it" seam this component's own
`resolveRichTextDocument` pattern follows, see `space-ui-architecture`. File:line
references point at `~/Documents/Development/ZanixLibraries/space-ui` — read
the real code there before assuming this summary is still accurate.

## Golden rule (token savings)

- Confirm a specific tag's target component from this skill's table rather
  than reading `tags.ts` in full for a routine content question.
- The ICU vs. Markdown mutual-exclusivity rule (below) is a one-fact check —
  apply it directly rather than tracing through the formatter each time.

## Two content formats, one explicit switch

`contentFormat: 'icu' | 'markdown'` (default `'icu'`) — no sniffing by
content or origin. **`'markdown'` mode deliberately never runs `content`
through `formatMessage`/ICU first**: ICU uses `{...}` for its own syntax,
and real Markdown (a fenced code block showing JSON, for one) commonly
contains literal braces that ICU parsing would otherwise misinterpret
before Markdown ever saw them. Don't route Markdown content through the ICU
formatter "just in case" — it's an active source of corruption for content
containing braces, not a safety net.

`RichText` requires `<IntlProvider>` uniformly, even in `'markdown'` mode
(which doesn't actually use the formatter) — one invariant to document
instead of a conditional one, and React's own Hooks rules make `useIntl()`
awkward to call conditionally regardless.

## ICU mode: built on `formatRichText`, not a hand-rolled tag parser

`Formatter.formatRichText<T>` is a public, generic addition to the same
`Formatter` interface `formatMessage` already lives on, built directly on
`IntlShape.formatMessage`'s own generic rich-text overload. `tags`/`values`
merge into ONE record before the real call — a pure call-site convenience,
not a second semantics.

### Tags — what's built in

Structural/text tags (`p`, headings, emphasis, lists, etc.) render plain
HTML directly. A smaller set targets a real component:

| Tag | Target |
| --- | --- |
| `a` | `Link` |
| `img` | `Image` |
| `video` | `Video` |
| `btn` | `Button` |
| `icon` | `CatalogIcon` (named icons — no `href`/`viewBox` to manage at the call site) |
| `sn` | `SocialNetworks` |
| `ibtn` | `ImgButton` |
| `ifrm` | `IFrame` |
| `sus` | `Skeleton` (loading-placeholder intent) |

Every component-targeting tag inherits that component's own `data-space-ui`
hook, never `"richtext"` — see `space-ui-component-patterns`'s composed-vs-
reimplemented rule, which this follows exactly.

**Two tags with no target in this package**: `page`/`lc` (page-level
composition is `@zanix/space`'s job, per `space-ui-architecture`'s ownership
map — out of scope here regardless of any future component) and `menu` (no
renderer-agnostic `render.ts` factory exists for `Menu` — it needs real
per-renderer hooks — and no consumer evidence has asked for it inside rich
text specifically). Don't assume adding one of these tags is a small
addition; both are structurally out of scope or blocked on a real
architectural gap, not a missing implementation detail.

### Population (`<props>key=val</props>`): a typed sentinel, never a string round-trip

ICU tags have no attribute syntax, so a nested `<props>` tag is how extra
props reach an enclosing tag. **This is implemented as a typed sentinel
value** (an unexported `Symbol`-tagged object, never a shape a caller's own
chunk could accidentally satisfy), carried through the chunks array as real
data — **not** a stringified marker that gets regex-matched and re-parsed
later. A string-round-trip implementation of this exact mechanism produced
two real, confirmed bugs in a predecessor system: a literal `$` character
breaking the matching regex, and plain text silently misparsed as a
querystring for component-type tags. The typed-sentinel approach structurally
can't reproduce either failure mode — if a future change to population
reintroduces string-marker matching for any reason, treat that as a
regression back into both bugs' failure class, not a neutral implementation
choice.

`extractRichTextProps` is the one piece of the population mechanism exported
publicly — a custom tag passed through `RichText`'s own `tags` prop
participates in population the same uniform way every built-in tag does.
Merge policy: `className` concatenates, a plain `style` object shallow-
merges, everything else is last-write-wins.

## Markdown mode: a pure-AST parser, zero `preact/compat`

`markdown-to-jsx`'s own `/markdown` subpath — a pure markdown→AST parser
with **zero React import at runtime** (verified against its actual built
JS, not its React-flavored `.d.ts` or root/`/react` entrypoint) — is walked
by hand via `h`, the same "parameterized by `h`" pattern
`createCatalogIcon` uses (see `space-ui-icons`). This is why Markdown
renders through Preact with no `preact/compat` involved anywhere — enforced
by a real dependency-boundary test (`mod.ts`/`mod-preact.ts` both reach
`markdown-to-jsx`, but never through `preact/compat`), not just asserted.

Link/image URLs in Markdown route through the **same** `parsePropsQuery`
mechanism the ICU side's `<props>` tag uses — one implementation, not two
parallel ones for the two content formats.

**Real, disclosed v1 scope limit, not a hidden gap**: tables, footnotes,
GFM tasks, frontmatter, and raw HTML/JSX blocks render as nothing, never a
crash. Covers the common node kinds a real CMS/docs document needs
(paragraphs, headings, emphasis, links, images, code, lists, blockquotes,
breaks).

## Loading content: `resolveRichTextDocument`, never fetch-in-component

`resolveRichTextDocument` (`RichText/resolve.ts`) is a standalone,
`fetch`-based async resolver, called from a `@zanix/space` `loader` — by
the time content reaches `RichText`, it's already resolved. This mirrors
`StructuredData`'s own "component plus an adjacent pure/async resolver"
precedent exactly, not a new abstraction. This is the concrete instance of
`space-ui-architecture`'s seam 7 ("presents data, never owns it") applied to
document loading specifically: a component doing its own client-side fetch
for its primary content is exactly the shape that seam forbids, and doing
so also breaks seam 6 (deterministic first render) the moment the fetch is
client-only. A failed load rejects with a real error at the resolver level,
never something the component itself needs to render a fallback for.

## Checklist before touching RichText or adding a tag

- [ ] **Whenever a brand-new component ships elsewhere in this package**, was
      it evaluated as a RichText tag candidate — not skipped by default? Only
      presentational/content-embeddable components belong (a link, image,
      icon, styled action trigger) — anything that renders sensibly inline
      inside a translated ICU message. A stateful, data-driven interactive
      control (needs `options`/`value`/`onChange` wired to real app state,
      not translatable message content) never does — the same reason none of
      `RadioGroup`/`Tabs`/`Accordion`/`Combobox`/`Modal`/`Select` are tags
      today, not an oversight to fix. Confirm which bucket a new component
      falls in explicitly, don't just leave it untagged by inertia.
- [ ] Does the tag render plain HTML directly (gets `"richtext"`), or a real
      component (inherits that component's own hook, never `"richtext"`)?
- [ ] If adding a component-targeting tag, does a renderer-agnostic
      `render.ts` factory actually exist for that component? If not (like
      `Menu` today), the tag isn't a small addition — flag it rather than
      building a one-off shortcut.
- [ ] Does any population/props mechanism route through the typed sentinel,
      never a stringified marker re-parsed via regex?
- [ ] Is Markdown content kept out of the ICU formatter entirely — never
      run through `formatMessage` "just to be safe"?
- [ ] Does new content-loading logic go through a standalone resolver called
      from a `loader`, never a fetch inside the component itself?

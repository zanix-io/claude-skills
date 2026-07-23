# Zanix - Claude Skills

Reusable [Claude Code Skills](https://code.claude.com/docs/en/skills) for working across the Zanix
ecosystem — backend libraries and services today, with room to grow into frontend and microapp
projects as they adopt Claude Code.

## What's here

| Skill                        | Use it for                                                                                     |
| ----------------------------- | ------------------------------------------------------------------------------------------------- |
| `complete-test-coverage`     | Auditing and closing real test-coverage gaps in any stack (Deno, Node, Python, Go...) without wasting tokens on dead/unreachable code. |
| `jsdoc-jsr-audit`            | Auditing existing JSDoc for accuracy (not just presence) and driving `deno doc --lint` to zero in Deno/JSR **packages**. |
| `docs-readme-audit`          | Making a Deno/JSR **package**'s README, `docs/`, and CHANGELOG complete, coherent, and professional in one exhaustive pass. |
| `zanix-server-conventions`   | Writing/reviewing **application** code on top of `@zanix/server` (handlers, interactors, providers, middlewares, sockets, jobs), grounded in real production Zanix microservices. |

`jsdoc-jsr-audit` and `docs-readme-audit` are scoped to publishable **libraries** (anything with a
`deno.json(c)` `"exports"` field) — they don't apply to a deployed app/microservice that isn't
published as a package. `complete-test-coverage` and `zanix-server-conventions` aren't library-only:
the former works on any project with tests, the latter on any app consuming `@zanix/server`,
published or not.

Each skill is self-contained in its own `SKILL.md` under `.claude/skills/<name>/`, distilled from
real, hands-on sessions — not written speculatively. They encode what actually needed fixing, what
turned out to be a false alarm, and the token-saving habits (batch analysis, automated
link/symbol/anchor checks instead of re-reading files, `git stash` baselines) that made those
sessions efficient.

## Using these skills in a project

Pick whichever fits your setup:

- **Symlink individual skills** into a project's own `.claude/skills/`:

  ```bash
  ln -s /path/to/claude-skills/.claude/skills/docs-readme-audit .claude/skills/docs-readme-audit
  ```

- **Point Claude Code at this whole repo** with `--add-dir`, which auto-loads everything under its
  `.claude/skills/`:

  ```bash
  claude --add-dir /path/to/claude-skills
  ```

- **Package as a plugin** and distribute via a marketplace, if you're managing this across many
  repos/teams.

Once available, invoke a skill directly (e.g. `/docs-readme-audit`) or let Claude pick it up
automatically when a request matches its `description`.

## Contributing

These skills are meant to keep evolving: when a session uncovers a new pattern, gotcha, or
token-saving trick worth generalizing, fold it into the relevant `SKILL.md` (or add a new skill)
rather than letting it stay a one-off prompt in a single project.

## License

MIT — see [LICENSE](./LICENSE).

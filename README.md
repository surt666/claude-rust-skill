# rust-style

A shareable Claude Code skill that teaches agents to write **types-first, closures-as-ports Rust** — the same style whether a human or an agent is at the keyboard.

## What the skill teaches

- **Make illegal states unrepresentable** — sum types with per-variant data, newtypes over primitives, `strum` for string⇄enum.
- **Parse, don't validate** — constructors that return `Result<T, E>` and hand back a value whose type proves it's valid.
- **Closures as ports, not traits** — business logic takes its effects (DB, clock, external calls) as `async` closure parameters, so the logic layer is pure and unit-tested with no DB, mocks, or trait objects. Includes the decision rule for when a trait *does* earn its place.
- **Join independent awaits** — futures that don't depend on each other run together via `try_join!` / `join!` / `try_join_all`, not one `.await` after another.
- **Typed builders** (`typed-builder`), **`thiserror` in the domain / translate at the edge**, and **generics/`impl Trait` over `dyn`**.
- A quick-reference table, common mistakes, and a red-flags checklist.

The skill activates automatically when an agent is writing or reviewing Rust.

## Install (team)

Point Claude Code at this repo as a marketplace, then install the plugin:

```
/plugin marketplace add <your-org>/claude-rust-skill
/plugin install rust-style@rust-style-marketplace
```

Or add it to a project's `.claude/settings.json` so everyone on the repo gets it automatically:

```json
{
  "extraKnownMarketplaces": {
    "rust-style-marketplace": {
      "source": { "source": "github", "repo": "<your-org>/claude-rust-skill" }
    }
  },
  "enabledPlugins": ["rust-style@rust-style-marketplace"]
}
```

## Layout

```
.claude-plugin/marketplace.json     # marketplace manifest (lists the plugin)
plugins/rust-style/
  .claude-plugin/plugin.json        # plugin manifest
  skills/writing-rust/SKILL.md      # the skill itself
```

## License

MIT — adapt freely.

# Gleam — Claude Code Plugin

A [Claude Code](https://claude.ai/code) plugin that injects Gleam language
knowledge into the assistant's context. When active, Claude can write correct
Gleam code, follow official conventions, use the right stdlib APIs, and navigate
the Erlang/JS ecosystem — without hallucinating syntax or reaching for
non-existent functions.

## Installation

```sh
claude plugin add https://github.com/ghivert/gleam-claude-code
```

The plugin is enabled by default. All knowledge skills load automatically for
any project where the plugin is active.

## What's included

- `gleam-syntax` — Full language reference — types, pattern matching, pipelines,
  generics, `use` expressions, bit arrays, FFI annotations
- `gleam-conventions` — Official naming rules, module structure, anti-patterns,
  and idiomatic code patterns from the Gleam docs
- `gleam-errors` — `Result`/`Option` module APIs, `use`-expression chaining,
  error type design, `panic`/`let assert`/`todo` guidance
- `gleam-types` — Opaque types, phantom types for compile-time state tracking,
  `Dynamic` decode as a boundary validation strategy
- `gleam-project` — Directory layout, module system, import paths, `gleam.toml`
  reference
- `gleam-testing` — gleeunit runner, `assert`/`should.*` assertions, birdie
  snapshots, qcheck property tests, fixture patterns
- `gleam-ffi` — `@external` syntax, Erlang and JS FFI, module name mangling,
  `gleam_erlang` helpers, `Dynamic`/decode, bundler integration
- `gleam-ecosystem` — Notable stdlib and third-party packages,
  `gleam_javascript` API (Promise/Array/Symbol), community links
- `gleam-otp` — Actors, supervisors, subjects, selectors, process monitoring,
  named processes, timers
- `gleam-lustre` — Lustre frontend framework — MVU architecture,
  HTML/events/effects API, components, SSR
- `gleam-wisp` — Wisp + Mist web stack — routing, middleware, JSON, form data,
  WebSockets, SSE
- `gleam-redraw` — Redraw React bindings — components, hooks, event handling,
  React interop
- `gleam-cli` — Gleam CLI commands — build, test, run, format, deps, publish,
  export

## Usage

Most skills are passive — they load silently and inform every response. The CLI
skill can also be invoked directly:

```
/gleam-cli build
/gleam-cli test
```

## Contributing

Skills live under `skills/<topic>/SKILL.md`. Each file has YAML frontmatter
(`name`, `description`, `user-invocable`) followed by markdown that gets
injected into Claude's context.

To add or update knowledge, edit the relevant skill file — or create a new
`skills/<topic>/SKILL.md` following the existing format. Keep the table in
`CLAUDE.md` in sync.

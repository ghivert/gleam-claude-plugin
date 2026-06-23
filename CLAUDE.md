# Gleam — Claude Code Plugin

This is a **Claude Code plugin** that provides Gleam language knowledge to
Claude Code as injectable skills.

## Structure

```
.claude-plugin/plugin.json   # plugin manifest
skills/
  <topic>/SKILL.md           # one skill per topic
```

Each `SKILL.md` has YAML frontmatter (`name`, `description`, `user-invocable`)
followed by markdown content that gets injected into Claude's context when the
skill is active.

## Adding knowledge

**Always create or update a skill file** — never put Gleam knowledge directly in
`CLAUDE.md`. When new Gleam documentation, conventions, or patterns need to be
ingested:

1. Find the right existing skill under `skills/`, or create a new
   `skills/<topic>/SKILL.md`.
2. Follow the frontmatter format of existing skills.
3. Set `user-invocable: false` for reference/knowledge skills, `true` for action
   skills (like `gleam-cli`).

## Existing skills

**Keep this table up to date after every modification** — add rows for new
skills, remove rows for deleted skills, update names if renamed.

| Directory             | Skill name          | Content                                                                                                                                                                                                                |
| --------------------- | ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `skills/cli/`         | `gleam-cli`         | Gleam CLI commands — build, test, run, hex, export, lsp (user-invocable)                                                                                                                                               |
| `skills/conventions/` | `gleam-conventions` | Official conventions, patterns, anti-patterns                                                                                                                                                                          |
| `skills/errors/`      | `gleam-errors`      | Error handling — Result/Option APIs, use-expression chaining, error type design, panic/assert guidance                                                                                                                 |
| `skills/ecosystem/`   | `gleam-ecosystem`   | Notable stdlib/ecosystem packages, gleam_javascript API (Promise/Array/Symbol), and community links                                                                                                                    |
| `skills/ffi/`         | `gleam-ffi`         | FFI — @external syntax, Erlang/.erl files, module name mangling, gleam_erlang helpers (atom/charlist/process), Dynamic/decode, JS bundler integration (Vite/esbuild/Node), Gleam↔JS type mapping, multi-target gotchas |
| `skills/lustre/`      | `gleam-lustre`      | Lustre web framework                                                                                                                                                                                                   |
| `skills/otp/`         | `gleam-otp`         | OTP — actors (gen_server), static/factory supervisors, subjects, selectors, monitors, named processes, timers                                                                                                          |
| `skills/project/`     | `gleam-project`     | Project structure, modules, testing, and gleam.toml reference                                                                                                                                                          |
| `skills/redraw/`      | `gleam-redraw`      | Redraw React bindings framework                                                                                                                                                                                        |
| `skills/syntax/`      | `gleam-syntax`      | Full language syntax reference                                                                                                                                                                                         |
| `skills/testing/`     | `gleam-testing`     | Testing — gleeunit runner, assert/should assertions, birdie snapshots, qcheck property tests, fixtures                                                                                                                 |
| `skills/types/`       | `gleam-types`       | Type system patterns — opaque types for encapsulation, phantom types for compile-time state tracking, Dynamic decode as boundary validation                                                                            |
| `skills/wisp/`        | `gleam-wisp-mist`   | Wisp web framework + Mist HTTP server                                                                                                                                                                                  |

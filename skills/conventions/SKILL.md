---
name: gleam-conventions
description:
  Gleam conventions, patterns, and anti-patterns — official style rules for
  naming, modules, error handling, and code structure
user-invocable: false
---

# Gleam Conventions, Patterns, and Anti-patterns

Source: https://gleam.run/documentation/conventions-patterns-and-anti-patterns/

## Conventions

- **Avoid unqualified importing of functions and constants** — use qualified
  syntax (`module.function`). Types and record constructors may be unqualified
  when readability isn't compromised.
- **Annotate all module functions** — every public function needs explicit type
  annotations for parameters and return values.
- **Use `Result` for fallible functions** — functions that might fail return
  `Result`, not `Option` or panic. Use `Result(a, Nil)` when failure holds no
  additional data.
- **Use singular for module names** — `import app/user` not `import app/users`.
- **Treat acronyms as single words** — write `json` not `JSON`. Prevents
  unexpected naming in generated BEAM code.
- **Conventional conversion function naming** — use `x_to_y`. When module name
  matches the type, omit repetition: `identifier.to_string(id)` not
  `identifier.identifier_to_string(id)`.
- **Conventional fallible function naming** — name result-returning functions by
  domain. Use `try_` prefix only when no better domain-specific name exists.
- **Use the core libraries** — depend on `gleam_stdlib`, `gleam_time`,
  `gleam_json`, etc. rather than reimplementing them.
- **Keep dev tool config in gleam.toml** — under `[tools.$TOOL_NAME]` sections,
  not separate files.
- **Use the correct source directories** — `src` for app code, `test` for tests,
  `dev` for development helpers.

## Patterns

- **Design descriptive errors** — error type variants should describe
  business-domain issues with context fields. Nest lower-level errors to
  preserve debugging info.
- **Comment liberally** — document both what and why, for future readers and bug
  investigation.
- **Make invalid states impossible** — use custom types to encode business rules
  so invalid data can't be constructed.
- **Replace bools with custom types** — semantic meaning, prevents confusion,
  simplifies future expansion beyond two states.
- **The sans-io pattern** — design API clients with paired functions: one
  constructing requests, one parsing responses. Decouples the library from HTTP
  implementations.
- **The builder pattern** — chain methods transforming records with optional
  fields, starting from required parameters. Provide convenience functions for
  common configs.

## Anti-patterns

- **Abbreviations** — use full names; they improve readability across diverse
  teams.
- **Fragmented modules** — avoid premature module splitting. Single
  well-designed modules beat scattered functionality requiring multiple imports.
- **Panicking in libraries** — libraries must return `Result`. Panicking removes
  user control over error handling.
- **Global namespace pollution** — place package modules in directories matching
  the package name to prevent compilation conflicts.
- **Namespace trespassing** — don't place modules in directories owned by other
  packages.
- **Grouping by design pattern** — structure modules by business domain, never
  by abstract patterns like "controllers" or "functors".
- **Check-then-assert** — avoid checking a condition then asserting. Use pattern
  matching and result combinators instead.
- **Using `Dynamic` with FFI** — create specific types for external values
  instead of `Dynamic`, which accepts anything and causes runtime errors.
- **Match all variants** — avoid catch-all patterns; exhaustiveness checking
  aids refactoring. Match every variant explicitly.
- **Category theory overuse** — Gleam lacks ergonomics for complex abstractions.
  Solve specific problems with concrete, understandable code.
